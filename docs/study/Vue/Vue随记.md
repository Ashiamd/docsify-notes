# Vue随记

> 最近做毕设，后端Springboot+Flink已经将SlopeOne推荐算法整合完毕了，需要先做一下前端，决定用比较熟的Vue。=> 之后毕设要是结束，有时间整理的话，会把项目开源到github上。
>
> 这里再开个随记markdown笔记文件。

## 1.Vue模版

> 同学做作业的时候，偶然发现了一个Vue模版，我觉得挺不错的，分享一下。

1. [VSCode一键生成.vue模版](https://zhuanlan.zhihu.com/p/136906175)

   ```vue
   <template>
   <div>
     <PageHeaderLayout>
   
     </PageHeaderLayout>
     </div>
   </template>
   
   <script>
     // 这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
     import PageHeaderLayout from '@/layouts/PageHeaderLayout'
     import ApeDrawer from '@/components/ApeDrawer'
     import ModalDialog from '@/components/ModalDialog'
     import ApeUploader from '@/components/ApeUploader'
     import ApeEditor from '@/components/ApeEditor' 
     import { mapGetters } from 'vuex'
   
     export default {
       components: {
         PageHeaderLayout,
         ApeDrawer,
         ModalDialog,
         ApeUploader,
         ApeEditor
       },
       // 定义属性
       data() {
         return {
   
         }
       },
       // 计算属性，会监听依赖属性值随之变化
       computed: {
         ...mapGetters(['userPermissions','buttonType'])
       },
       // 监控data中的数据变化
       watch: {},
       // 方法集合
       methods: {
   
       },
       // 生命周期 - 创建完成（可以访问当前this实例）
       created() {
   
       },
       // 生命周期 - 挂载完成（可以访问DOM元素）
       mounted() {
   
       },
       beforeCreate() {}, // 生命周期 - 创建之前
       beforeMount() {}, // 生命周期 - 挂载之前
       beforeUpdate() {}, // 生命周期 - 更新之前
       updated() {}, // 生命周期 - 更新之后
       beforeDestroy() {}, // 生命周期 - 销毁之前
       destroyed() {}, // 生命周期 - 销毁完成
       activated() {}, // 如果页面有keep-alive缓存功能，这个函数会触发
     }
   </script>
   
   <style lang='stylus' scoped>
   
   </style>
   ```

   

