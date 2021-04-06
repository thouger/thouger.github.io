```rust
enum Poll<T> {
    Ready(T),
    Pending
}

trait Future {
    type Output;

    fn poll(&mut self, ctx: &Context) -> Poll<Self::Output>;
}

#[derive(Default)]
struct MyFuture {
    count: u32,
}

/*
这里的ctx是在run被传入Context类进去
 */
impl Future for MyFuture {
    type Output = i32;

    fn poll(&mut self, ctx: &Context) -> Poll<Self::Output> {
        match self.count {
            3 => Poll::Ready(3),
            _ => {
                self.count += 1;
                ctx.waker().wake();
                Poll::Pending
            }
        }
    }
}

struct AddOneFuture<T>(T);
impl<T> Future for AddOneFuture<T>
    where
        T: Future,
        T::Output: std::ops::Add<i32, Output = i32>,
{
    type Output = i32;

    fn poll(&mut self, ctx: &Context) -> Poll<Self::Output> {
        match self.0.poll(ctx) {
            Poll::Ready(count) => Poll::Ready(count + 1),
            Poll::Pending => Poll::Pending,
        }
    }
}

/*
thread_local!适用于static mut(全局静态可变变量),释放资源需要手动释放,不可以用std::ops::Drop,
    无论访问还是Drop,都需要用xxx.with|x|{}的方式访问,这里用来声明一个全局公共静态变量RefCell,主要用来多次借用可变引用
RefCell被封装到只能在闭包中使用,所以借用完可变引用释放完又可以借用了
    RefCell的特点是可借用可变引用和不可变引用,可多次借用不可变引用,但还是不能借用多次可变借用,但是配合thread_local!.with就可以
这里的NOTIFY贯穿全文,一开始复制ture,然后在run函数里开始循环,开始循环后被复制false
 */
use std::cell::RefCell;
thread_local!(static NOTIFY: RefCell<bool> = RefCell::new(true));

/*
相同的声明周期,只会选择最小最短的那个来用,哪个最先被销毁,生命周期就是哪个
struct定义引用,必须加生命周期
 */
struct Context<'a> {
    waker: &'a Waker,
}
impl<'a> Context<'a> {
    fn from_waker(waker: &'a Waker) -> Self {
        Context { waker }
    }

    fn waker(&self) -> &'a Waker {
        &self.waker
    }
}

struct Waker;
impl Waker {
    fn wake(&self) {
        NOTIFY.with(|f| *f.borrow_mut() = true)
    }
}


fn run<F>(mut f: F) -> F::Output
    where
        F: Future,
{
    NOTIFY.with(|n| loop {
        if *n.borrow() {
            *n.borrow_mut() = false;
            // 这里Context::from_waker就是创建一个带有Waker引用的Context类,然后传入AddOneFuture的poll里面
            let ctx = Context::from_waker(&Waker);
            /*
                f.poll等于AddOneFuture里面调用了my_future的poll
                my_future的poll计算my_future.count,当3的时候返回Poll::Ready(3),当不是3的时候my_future.count+1并/将NOTIFY值设为true/返回Poll::Pending
                AddOneFuture判断my_future的poll返回的是Poll::Ready还是Poll::Pending,如果是Ready就将返回的count+1,同时将这个结果(无论Ready还是Pending)都返回到run函数这里
                所以这里就是判断AddOneFuture的my_future的poll返回的是Ready还是Pending

                这里的NOTIFY值,似乎没有很大的帮助,就是在初始化的时候设为true,使循环开始,然后循环一开始就将设为false,在my_future里面又设置为true
             */
            if let Poll::Ready(val) = f.poll(&ctx) {
                return val;
            }
        }
    })
}

/* 继承关系
枚举Poll下面有Ready<T>和Pending

接口Future下有一个类型Output,一个poll方法
一个结构体MyFuture,只有一个计数结果
一个结构体AddOneFuture,没有任何东西
MyFuture和AddOneFuture继承FuTure,实现Output和Poll

一个结构体Context,下面有Walker
一个类Context,下面有walker和from_waker方法

一个结构体Waker,下面有wake方法
 */
fn main() {
    let my_future = MyFuture::default();
    println!("Output: {}", run(AddOneFuture(my_future)));
}
```