# Vue路由懒加载loading

VueRouter并不支持这种写法

```javascript
const instance = new Router({
	base: process.env.projectMeta.ROUTER_BASE,
	mode: 'history',
	routes: [
		{
			path: '/',
			name: 'post',
			component:()=>({
				component:import('@/views/post'),
				loading:Skeleton
			}),
	],
});
```

{% embed url="https://github.com/vuejs/vue-router/issues/2830" %}

在VueRouter的github中找到了这条issue,根据里边提供的方案 [https://github.com/chrisvfritz/vue-enterprise-boilerplate/blob/master/src/router/routes.js](https://github.com/chrisvfritz/vue-enterprise-boilerplate/blob/master/src/router/routes.js)，针对自己的业务做了一些改造

```javascript
export  const  lazyLoadView=(loadComponent)=>async () =>{
	//在加载中或者加载错误时可以有回退动画，但是加载成功后替换成页面组件的过程中不需要动画
	const props=Vue.observable({
		loading:true
	});
	const AsyncComponent  = () => ({
		component: loadComponent().then(async c=>{
			props.loading=false;
			await Vue.nextTick();
			return c;
		}),
		loading: {
			functional: true,
			render (h,{data}) {
				return h(Skeleton,{
					...data,
					props:{
						loading:props.loading,
						...data.props
					},
				});
			},
		},
		delay: 400,
		error: {
			functional: true,
			render (h,{data}) {
				return h(Skeleton,{
					...data,
					props:{
						error:true,
						...data.props
					},
				});
			},
		},
		timeout: 30000,
	});
	return {
		functional: true,
		render (h,{data}) {
			return h(AsyncComponent,data);
		},
	};
};

const instance = new Router({
	base: process.env.projectMeta.ROUTER_BASE,
	mode: 'history',
	routes: [
		{
			path: '/',
			name: 'post',
			component:lazyLoadView(()=>import('@/views/post'))
		},
	],
});
```



