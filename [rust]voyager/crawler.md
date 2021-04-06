```rust
use crate::Scraper::Scraper;
use std::sync::Arc;
use std::pin::Pin;
use crate::Response::Response;
use std::future::Future;
use anyhow::Result;
use crate::CrawlerConfig::CrawlerConfig;
use crate::requests::{response_info, QueuedRequestBuilder};
use futures::future::ok;
use std::task::{Context, Poll};
use crate::CrawlResult::CrawlResult;
use std::collections::VecDeque;
use futures::{FutureExt, Stream};
use crate::domain::{DomainListing, BlockList, AllowList, AllowListConfig};
use reqwest::IntoUrl;

//#定义了Stats结构体,里面有request_count和response_count
#[derive(Debug, Clone, Copy, Default)]
pub struct Stats {
    pub request_count: usize,
    pub response_count: usize,
}

type CrawlRequest<T> = Pin<Box<dyn Future<Output = Result<Response<T>>>>>;
type OutputRequest<T> = Pin<Box<dyn Future<Output = Result<Option<T>>>>>;


pub struct Crawler<T:Scraper>{
    client:Arc<reqwest::Client>,
    pub(crate) current_depth:usize,
    in_progress_crawl_requests:Vec<CrawlRequest<T::State>>,
    queued_results:VecDeque<CrawlResult<T>>,
    in_progress_complete_requests:Vec<OutputRequest<T::Output>>,
    pub stats: Stats,
    list: DomainListing<T::State>,
}
/*
Scraper的Crawler类将结构体定义的赋值
    {
    创建了一个reqwest::Client多线程版本,用于多线程get,调用方式是直接self.client
    Arc类型是Rc的多线程版,用于跨线程共享对象(所有权和引用),因为默认情况下共享引用是不允许mut的，因此是无法得到Arc内部里面的可变引用的,想mut可以考虑使用Mutex,RwLock
    与Rc不同的是，Arc使用原子操作进行引用计数。这意味着它是线程安全的。缺点是，原子操作比普通内存访问更昂贵。如果你不在线程之间共享引用计数的分配，可以考虑使用Rc来降低开销。
    }
 */
impl<T:Scraper> Crawler<T>{
    pub fn new(config: CrawlerConfig) -> Self{
        let client = Arc::new(Default::default());
        let list = {
            let block_list = BlockList::new(
                config.disallowed_domains,
                Arc::clone(&client),
                config.respect_robots_txt,
                config.skip_non_successful_responses,
                config.max_depth.unwrap_or(usize::MAX),
                config
                    .max_requests
                    .unwrap_or(CrawlerConfig::MAX_CONCURRENT_REQUESTS),
            );
            DomainListing::BlockList(block_list)
        };
        Self{
            client,
            current_depth:0,
            in_progress_crawl_requests:Default::default(),
            in_progress_complete_requests: Default::default(),
            queued_results: Default::default(),
            stats: Default::default(),
            list
        }
    }
}

/*
特别的的Crawler,限制于T是三种情况:
    1.T是Scraper,又是属于Unpin类型的静态
    2.T的State是要Unpin和send、Sync类型的静态
    3.T的Output是Unpin类型
    限制T的三种情况,应该是用到poll_next才需要限制的
一共实现了fn crawl方法、fn complete方法、fn visit方法、fn poll方法
 */
impl<T> Crawler<T>
    where
        T: Scraper + Unpin + 'static,
        <T as Scraper>::State: Unpin + Send + Sync + 'static,
        <T as Scraper>::Output: Unpin,{

    pub fn visit_with_state(&mut self, url: impl IntoUrl, state: T::State) {
        self.request_with_state(self.client.request(reqwest::Method::GET, url), state)
    }

    pub fn visit(&mut self, url: impl IntoUrl) {
        self.request(self.client.request(reqwest::Method::GET, url))
    }

    pub fn request(&mut self, req: reqwest::RequestBuilder) {
        self.queue_request(req, None)
    }

    pub fn request_with_state(&mut self, req: reqwest::RequestBuilder, state: T::State) {
        self.queue_request(req, Some(state))
    }

    fn queue_request(&mut self, request: reqwest::RequestBuilder, state: Option<T::State>) {
        let req = QueuedRequestBuilder {
            request,
            state,
            depth: self.current_depth + 1,
        };
        if let Err(err) = self.list.add_request(req) {
            self.queued_results
                .push_back(CrawlResult::Crawled(Err(err.into())))
        }
    }


    /*
    fn crawl函数定义i了TCrawlFunction,TCrawlFuture两个泛型,以及一个TCrawlFunction的fun形参
        TCrawlFunction定义了fun是一个返回TCrawlFuture,并且只能执行一次的闭包request::Client
        TCrawlFuture是限制fun的返回值是一个包含Response和State的Future<Result>
        暂不知道是在哪里调用的

        函数首先将现在的深度+1
        然后将self.client引用从多线程中取出转化为TCrawlFunction
        创建一个pin的闭包,获得status,url,headers,text
        生成装有depth,request_url,response_url,text,state的Response
        放进in_progress_crawl_requests
    */
    pub fn crawl<TCrawlFunction, TCrawlFuture>(&mut self, fun: TCrawlFunction)
        where
            TCrawlFunction: FnOnce(Arc<reqwest::Client>) -> TCrawlFuture,
            TCrawlFuture: Future<Output = Result<(reqwest::Response, Option<T::State>)>> + 'static,
    {
        let depth = self.current_depth + 1;
        let fut = (fun)(Arc::clone(&self.client));
        let fut = Box::pin(async move{
            let (mut resp,state) = fut.await?;
            let (status,url,headers) = response_info(&mut resp);
            let text = resp.text().await?;

            Ok(Response{
                depth,
                request_url:url.clone(),
                response_url:url,
                response_status:status,
                text,
                state,
            })
        });
        self.in_progress_crawl_requests.push(fut)
    }

    /*
    poll_next函数实现一个Item类型以及poll_next函数
    Item:Result<T::Output>:
    fn poll,函数有两个参数,一个是用了pin固定自身可变的引用,一个是可变的Context引用
        {
            至于为什么要用pin将自身的可变引用固定,这样就不可以用过safe代码拿到&mut self了.原因是因为原本在future 0.1版本poll参数时&mut self的
            但是如果在poll代码块里面,万一调用到std::mem::swap,将自身的&mut self改变指向另一个引用,那么self本身就会丢失,因此需要用pin固定不给改变&mut Self
            为什么要用pin?Box、Rc、Arc等指针类型也可以调用到实例,但是这些指针类型会在safe代码中将&mut Self暴露出来,这样又会导致实例被移动,因此不暴露&mut Self,就是pin的目的
        }
        {cx:&mut Context:好像没有见到用到}
        {
            被Arc::clone复制了引用再强制转化为TCrawlFunction赋值给fut
            接fut又变成了用box创建了一个pin类型的智能指针,里面放着异步代码等待client的返回response和state,然后获得text,最后将这些全部返回,box最后被加到in_progress_crawl_requests的Vec<CrawlRequest<T::State>>里面

         }
     */
    pub fn poll(&mut self,cx:&mut Context<'_>) -> Poll<Option<CrawlResult<T>>>{
        loop {
            if let Some(result) = self.queued_results.pop_front(){
                return Poll::Ready(Some(result));
            }

            for n in (0..self.in_progress_complete_requests.len()).rev(){
                let mut request = self.in_progress_complete_requests.swap_remove(n);
                if let Poll::Ready(resp) = request.poll_unpin(cx){
                    match resp{
                        Ok(Some(output)) => {
                            self.queued_results.push_back(CrawlResult::Finished(Ok(output)));
                        }
                        Err(err) =>{
                            self.queued_results.push_back(CrawlResult::Finished(Err(err)));
                        }
                        _ => {}
                    }
                }else{
                    self.in_progress_complete_requests.push(request);
                }
            }

            for n in (0..self.in_progress_crawl_requests.len()).rev(){
                let mut request = self.in_progress_crawl_requests.swap_remove(n);
                if let Poll::Ready(resp) = request.poll_unpin(cx){
                    self.queued_results.push_back(CrawlResult::Crawled(resp));
                }else{
                    self.in_progress_crawl_requests.push(request);
                }
            }

            let mut busy = false;
            loop {
                match Stream::poll_next(Pin::new(&mut self.list),cx){
                    Poll::Ready(Some(resp)) => {
                        self.queued_results.push_back(CrawlResult::Crawled(resp));
                    }
                    Poll::Pending => {
                        busy = true;
                        break;
                    }
                    _ => break,
                }
            }

            if self.queued_results.is_empty(){
                if !busy
                    && self.in_progress_crawl_requests.is_empty()
                    && self.in_progress_complete_requests.is_empty()
                {
                    return Poll::Ready(None);
                }
                return Poll::Pending;
            }
        }
    }
}
```