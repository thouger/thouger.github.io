```rust
/*
定义scraper让用户继承,里面有Output和State类型,和一个scrape方法需要实现
State是一个Debug类型,是为了可以让println!输出
返回的是一个前面定义的装好Output的Option的Result类型
*/
use std::fmt;
use crate::Crawler::Crawler;
use crate::Response::Response;
use anyhow::Result;

pub trait Scraper:Sized{
    type Output;
    type State: fmt::Debug;
    fn scrape(
        &mut self,
        response: Response<Self::State>,
        crawler: &mut Crawler<Self>,
    ) -> Result<Option<Self::Output>>;
}
```