```rust
use scraper::{Html, Selector};
use url::Url;
use futures::channel::mpsc;
use futures::{executor, Future}; //standard executors to provide a context for futures and streams
use futures::executor::ThreadPool;
use futures::StreamExt;
use futures::stream::Stream;
use std::task::{Context, Poll};
use std::pin::Pin;
use anyhow::Result;
use std::sync::Arc;
use std::fmt;

/*
定义一个抓取器,用于被用户继承,它有以下成员:
    一个 Output类型,用来存储结果
    一个State,继承与fmt::Debug,用来表示response的状态码
    一个scraper方法,参数可变的self、repsonse和可变的Crawler
 */
pub trait Scraper:Sized{
    type Output;
    type State: fmt::Debug;
    fn scraper(
        &mut self,
        response:&mut Crawler<Self>,
    ) -> Result<Option<Self::output>>;
}

/*
定义了一个Collector结构体和两个类,一个主要返回类成员一个继承Stream
Collector结构体,里面有Crawler类和Scraper类成员,以及定义了一个泛型,规定Crawler是属于Scraper的,暂且不知道原因
Collector类,主要返回类成员和new函数
    fn new:传入Crawler和Scraper
Collector类,继承futures的Stream,实现它的Item类型和poll_next方法
    type Item:是一个result类型,接收Output类型
    fn poll_next:传入可变的自身类的引用pin(Collector)以及一个Context<'_>类型cx
        pin是为了调用crawler成员的poll函数
泛型T贯穿一切
 */
pub struct Collector<T> {
    crawler:Crawler<T>,
    scraper:T,
}
impl<T> Collector<T> {
    pub fn new() -> Self{
        Self{
            crawler:Crawler::new(config),
            scraper
        }
    }

}

impl Stream for Collector<T> {
    type Item = Result<T::Output>;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let pin = self.get_mut();
        loop{
            match pin.crawler.poll(cx) {
                Poll::Ready(Some(result)) => match result{
                    CrawlResult::Finished(Ok(output)) => Poll::Ready(Some(Ok(output))),
                    CrawlResult::Finished(Err(err)) => Poll::Ready(Some(Err(err))),
                    CrawlResult::Crawled(Ok(response)) => {
                        pin.crawler.current_depth = response.depth;
                        pin.crawler.stats.response_count=pin.crawler.stats.response_count.wrapping_add(1);

                        let output = pin.scraper.scrape(response,&mut pin.crawler);

                        pin.crawler.current_depth = 0;

                        match output{
                            Ok(Some(output)) => Poll::Ready(Some(Ok(output))),
                            Err(err) => Poll::Ready(Some(Err(err))),
                            _ => {}
                        }
                    }
                    crawlResult::Crawler(Err(err)) => Poll::Ready(Some(Err(err))),
                },
                Poll::Pending => Poll::Pending,
            }
        }
    }
}

/*
定义了Crawler结构体以及对应的类
Crawler结构体定义了一个Arc类型的client,以及unsize类型的current_depth
Crawler类分裂了两个,第一个是Crawler用来new的,一个是特别的的Crawler,
 */
pub struct Crawler<T>{
    client:Arc<reqwest::Client>,
    current_depth:unsize,
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
    pub fn new(config:CrawlerConfig) ->Self{
        Self{
            client:Arc::new(Default::default()),
            current_depth:0,
        }
    }
}
/*
特别的的Crawler,限制于T是三种情况:
    1.T是要即是Scraper,又是属于Unpin类型的静态
    2.T的State是要Unpin和send、Sync类型的静态
    3.
    限制T的三种情况,应该是用到poll_next才需要限制的
一共实现了fn crawl方法、fn complete方法、fn visit方法、fn poll方法
 */
impl<T> Crawler<T>
    where
        T: Scraper + Unpin + 'static,
        <T as Scraper>::State: Unpin + Send + Sync + 'static,
        <T as Scraper>::Output: Unpin,{

    /*

     */
    pub fn crawl<TCrawlFunction,TCrawlFuture>(&mut self,fun:TCrawlFunction)
    where
        TCrawlFunction:FnOnce(Arc<reqwest::Client>) -> TCrawlFuture,
        TCrawlFuture:Future<Output = Result<(reqwest::Response, Option<T::State>)>> + 'static,
    {
        let depth = self.current_depth+1;
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
    fn poll(){

    }
}

#[derive(Debug,Clone,Copy,Default)]
pub struct Stats{
    pub request_count:usize,
    pub repsonse_count:usize,
}

pub struct Response<T>{
    pub state:Option<T>,
}

enum CrawlResult<T: Scraper> {
    Finished(Result<T::Output>),
    Crawled(Result<Response<T::State>>),
}
```