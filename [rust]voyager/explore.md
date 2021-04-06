```rust
use std::collections::{HashMap, HashSet};
use reqwest::Url;
use scraper::Selector;
use voyager::Scraper::Scraper;
use voyager::Response::Response;
use voyager::Crawler::Crawler;
use anyhow::Result;
use voyager::Collector::Collector;
use voyager::CrawlerConfig::CrawlerConfig;
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    struct Explorer {
        /// visited urls mapped with all the urls that link to that url
        visited: HashMap<Url, HashSet<Url>>,
        link_selector: Selector,
        title_selector:Selector
    }
    impl Default for Explorer {
        fn default() -> Self {
            Self {
                visited: Default::default(),
                link_selector: Selector::parse("a").unwrap(),
                title_selector: Selector::parse("title").unwrap()
            }
        }
    }

    impl Scraper for Explorer {
        type Output = (usize, Url,String);
        type State = Url;

        fn scrape(
            &mut self,
            mut response: Response<Self::State>,
            crawler: &mut Crawler<Self>,
        ) -> Result<Option<Self::Output>> {
            if let Some(origin) = response.state.take() {
                self.visited
                    .entry(response.response_url.clone())
                    .or_default()
                    .insert(origin);
            }
            let mut title = "".to_string();
            response = match response.html().select(&self.title_selector).next() {
                Some(element) => {
                    title = element.inner_html();
                    response
                },
                _ => {
                    println!("{:}",response.request_url);
                    println!("{:}",response.response_url);
                    println!("{:}",response.text);
                    // fs::write("ttttttttttttt.text",response.text.clone());
                    // process::exit(0x0100);
                    response
                }
            };

            for link in response.html().select(&self.link_selector) {
                if let Some(href) = link.value().attr("href") {
                    if let Ok(url) = response.response_url.join(href) {
                        crawler.visit_with_state(url, response.response_url.clone());
                    }
                }
            }

            Ok(Some((response.depth, response.response_url,title)))
        }
    }

    let config = CrawlerConfig::default();
    let mut collector = Collector::new(Explorer::default(), config);

    /*
    程序的开始,实例化了Collector,调用里面的crawler的visit到这里
    然后将client.request包装起来访问,最后装成有返回结果和状态和和当前深度的QueuedRequest结构体,然后在domain解包把RequestBuilder拿出来进行发送
    最后又包装成QueuedRequest,放进VecDeque里

    第二步跳到继承future下Stream的Collector的poll_next函数,poll_next无限循环crawler的poll函数
    poll函数开头不断循环从queued_results里拿出结果出来,一开始是没有结果的,所以往下走跳到domain模板继承了Stream的DomainListing类的poll_next函数
            DomainListing的poll_next函数又跳到继承了Stream BlockList的poll_next函数里面,该函数调用Stream::poll_next(Pin::new(&mut pin.request_queue), cx)将request_queue等待请求的结构体QueuedRequest取出来,然后将结构体里面的东西放进get_response函数里面,get_response函数就是发送get请求判断成功与否,成功就获得status,url和headers和text/失败就返回自定义错误CrawlError::NosuccessResponse,函数饭后深度,请求链接和返回链接,状态码,网页内容和用户定义的state
            最后将这个Ok结果用Box::pin包装放进in_progress_crawl_requests,
            下面就开始迭代in_progress_crawl_requests里面的请求了,试图用poll_unpin解开cx如果为Poll::Ready的话,就返回Poll::Ready(Some(resp)),否则就降request放进in_progress_crawl_requests
            判断请求队列是否为空,为空返回Poll::Ready(None),否则返回Poll::Pending
        接下来从DomainListing的poll_next回到上一层poll函数判断是Ready还是Pending,在DomainListing函数是将第一个链接放进队列的,所以是Pending,最后判断结果队列是否为空,不为空返回Poll::Pending
    最后从crawler的poll返回到Collector的poll_next,判断结果是Pending,继续不断的循环到crawler.poll函数返回Ready为止,然后执行用户定义的scraper函数,获得返回值,这里的返回值也是用户定义的,返回Poll::Ready(Some(Ok(output)))
    在explore文件判断返回为Some了,开始输出output
    */
    collector.crawler_mut().visit("https://baike.baidu.com/");

    while let Some(output) = collector.next().await {
        if let Ok((depth, url,title)) = output {
            // println!("title is {} at depth: {} when Visited {}",title, url, depth);
        }
    }

    Ok(())
}

```