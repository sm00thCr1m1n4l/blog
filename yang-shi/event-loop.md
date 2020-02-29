# Event Loop

### 执行顺序

 macrotask =&gt; microtask queue =&gt; macrotask queue 

注意：microtask queue每执行一个之后会重新清空microtask queue

重复 2-3

举例

```javascript
console.log(1); //macrotask 1  

setTimeout(() => {
  console.log(2);//macrotask queue 1
  Promise.resolve().then(() => {
    console.log(3) // microtask queue 2-1
  });
});

new Promise((resolve, reject) => {
  console.log(4) //macrotask 2
  resolve(5)
}).then((data) => {
  console.log(data);  // microtask queue 1-1
})

setTimeout(() => {
  console.log(6);//macrotask queue 2
})

console.log(7); //macrotask 3

```

从上往下执行，首先执行macrotask 1，打印1

放入一个任务到macrotask queue（macrotask queue 1）

执行macrotask 2 打印4 \(new Promise的参数内部属于macrotask\) ，并加入一个任务到micrortask queue （microtask queue 1-1）

放入一个任务到macrotask queue（macrotask queue 2）

执行macrotask 3 打印 7 

macrotask 执行完毕，开始逐个执行microtask queue

执行microtask queue 1-1 打印5

microtask queue清空,开始执行macrotask queue

执行macrotask queue 1 打印2 并将console.log\(3\)放入下一次循环的microtask queue（microtask queue2-1）中

由于上一步在microtask queue中放进了一个任务，遂执行microtask queue 2-1 打印3

继续执行macrotask queue 2打印 6

结果

1，4，7，5，2，3，6



