# 后退时关闭页面弹窗

```javascript
/**
 * 弹窗控制mixin
 */
export const modalControl = {
	data () {
		return {
			visible:{}
		};
	},
	created(){
		const q={...this.$route.query}
		//刷新进入页面，移除__visible__字段
		delete q.__visible__
		this.$router.replace({
			name: this.$route.name,
			params:{
				...this.$route.params,
			},
			query:{
					...q,
			},
		})
		Object.keys(this.visible).forEach(field=>{
			this.$watch(`visible.${field}`, (val) => {
				if (!val && this.$route.query.__visible__ === field) {
					//弹窗关闭时后退
					this.$router.back();
				}else if(val){
					//弹窗开启时前进
					this.$router.push({
						name: this.$route.name,
						params:{
							...this.$route.params,
						},
						query: {
							...this.$route.query,
							__visible__: field
						}
					});
				}
			});
		})
	},
	methods: {

	},
	watch: {
		'$route.query.__visible__': {
			handler (val, before) {
				//导航前进后退时根据query.__visible__开关弹窗
				if (val) {
					this.$set(this.visible, val, true);
				} else if (before) {
					this.$set(this.visible, before, false);
				}
			}
		},
	}
};

```

