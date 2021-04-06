```rust
use futures::Stream;
use crate::Crawler::Crawler;
use anyhow::Result;
use std::task::{Poll, Context};
use std::pin::Pin;
use crate::CrawlResult::CrawlResult;
use crate::Scraper::Scraper;
use std::fmt;
use crate::CrawlerConfig::CrawlerConfig;

/*
Collector结构体,里面有Crawler类和Scraper类成员,以及定义了一个Scraper泛型
scraper是由用户实现的
crawler是定义了爬虫规则的类
*/
pub struct Collector<T:Scraper> {
    crawler:Crawler<T>,
    scraper:T,
}

/*
Collector类,限制T是Scraper和它是继承Debug的
    fn new:传入Crawler和Scraper
    fn crawler_mut:self的get方法

泛型T是Scraper,贯穿一切
 */
impl<T> Collector<T>
    where
        T:Scraper,
        <T as Scraper>::State:fmt::Debug,
{
    pub fn new(scraper:T,config:CrawlerConfig) -> Self{
        Self{
            crawler:Crawler::new(config),
            scraper,
        }
    }

    pub fn crawler_mut(&mut self) -> &mut Crawler<T> {
        &mut self.crawler
    }
}

/*
Collector类,继承futures的Stream,限制T是unpin和静态的scraper类型,T的State是,T的Output是Unpin类型
    type Item:是一个result类型,限制返回类型是Poll的Option的Result的Scraper的Output
    fn poll_next:传入可变的自身类的引用pin(Collector)以及一个Context<'_>类型cx
        p首先获得self的可变引用,然后调用Collector的Crawler的poll,将类型是可变Context可变引用传入进去,对返回值进行筛选
        # todo
*/
impl<T> Stream for Collector<T>
where
    T: Scraper + Unpin + 'static,
    <T as Scraper>::State: Unpin + Send + Sync + 'static,
    <T as Scraper>::Output: Unpin,{
    type Item = Result<T::Output>;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let pin = self.get_mut();
        loop{
            match pin.crawler.poll(cx) {
                Poll::Ready(Some(result)) => match result{
                    CrawlResult::Finished(Ok(output)) => return Poll::Ready(Some(Ok(output))),
                    CrawlResult::Finished(Err(err)) => return Poll::Ready(Some(Err(err))),
                    CrawlResult::Crawled(Ok(response)) => {
                        pin.crawler.current_depth = response.depth;
                        pin.crawler.stats.response_count=pin.crawler.stats.response_count.wrapping_add(1);

                        let output = pin.scraper.scrape(response,&mut pin.crawler);

                        pin.crawler.current_depth = 0;

                        match output{
                            Ok(Some(output)) => return Poll::Ready(Some(Ok(output))),
                            Err(err) => return Poll::Ready(Some(Err(err))),
                            _ => {}
                        }
                    }
                    CrawlResult::Crawled(Err(err)) => return Poll::Ready(Some(Err(err))),
                },
                Poll::Ready(None) => return Poll::Ready(None),
                Poll::Pending => return Poll::Pending,
            }
        }
    }
}
```