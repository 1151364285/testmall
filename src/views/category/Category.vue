<template>
  <div class="category">
    <!--    导航-->
    <nav-bar class="nav-bar">
      <div slot="center">商品分类</div>
    </nav-bar>
    <tab-control :titles="['综合', '新品', '销量']"
                 @tabClick="tabClick"
                 ref="tabControl1"
                 class="tab-control tab-control2"
                 v-show="isTabFixed"/>
    <div class="content">
      <category-menu :categories="categories"
                     @selectItem="selectItem" @showiid="showiid"></category-menu>
      <scroll id="tab-content"
              :data="[categoryData]"
              :probe-type="3"
              @scroll="contentScroll"
              ref="scroll">
        <div>
          <category-content :subcategories="showSubcategory" v-show="false"></category-content>
          <tab-control :titles="['综合','新品','销量']"
                       @tabClick="tabClick"
                       ref="tabControl2">
          </tab-control>
          <good-list :goods="showCategoryDetail"></good-list>
        </div>
      </scroll>
    </div>

    <back-top v-show="isShowBackTop" @click.native="backClick"></back-top>
  </div>
</template>

<script>
  import CategoryMenu from "./childComps/CategoryMenu";
  import CategoryContent from "./childComps/CategoryContent";
  import NavBar from "components/common/navbar/NavBar";
  import BackTop from "components/content/backTop/BackTop";
  import TabControl from "components/content/tabControl/TabControl";
  import Scroll from "components/common/scroll/Scroll";
  import GoodList from "components/content/goods/GoodsList";
  import {POP, NEW, SELL, TOP_DISTANCE} from "common/const";
  import {getCategory, getSubcategory, getCategoryDetail} from "network/category";
  import {itemListenerMixin, backTopMixin, tabControlMixin} from "common/mixin";
  export default {
    name: "Category",
    components: {
      CategoryMenu,
      NavBar,
      CategoryContent,
      // TabControl,
      // BackTop,
      Scroll,
      GoodList
    },
    mixins: [itemListenerMixin, backTopMixin, tabControlMixin],
    data(){
      return {
        categories: [],
        categoryData: {},
        scroll: null,
        isTabFixed: false,
        currentIndex: -1,
        currentType: POP,
        saveY: 0,
        tabOffsetTop: 0,
        iid:0
      }
    },
    computed: {
      showSubcategory() {
        if(this.currentIndex === -1) {
          return {}
          //console.log(this.categoryData[this.currentIndex]);
          // return this.categoryData[this.currentIndex]==undefined?{}:this.categoryData[this.currentIndex].subcategories
          return this.categoryData[this.currentIndex].subcategories
        }
        return this.categoryData[this.currentIndex].subcategories;
      },
      showCategoryDetail() {
        if(this.currentIndex === -1) {
          // console.log(this.categoryData[this.currentIndex].categoryDetail[this.currentType]);
          // return this.categoryData[this.currentIndex]==undefined?{}:this.categoryData[this.currentIndex].categoryDetail[this.currentType];
          return []
          //  return this.categoryData[this.currentIndex].categoryDetail[this.currentType];
        }
        //console.log(this.categoryData[this.currentIndex].categoryDetail);
        return this.categoryData[this.currentIndex].categoryDetail[this.currentType];//2222222
      }
    },
    created() {
      //请求分类数据
      this._getCategory();

    },
    mounted() {
      this.$bus.$on('gridViewImgLoad', () => {
        //refresh重新计算better-scroll,   this.$refs.scroll：组件创建完成才能执行后面函数
        this.refresh();
        this.tabOffsetTop = this.$refs.tabControl2.$el.offsetTop;
        this.tabOffsetLeft = this.$refs.scroll.$el.offsetLeft;
      })

    },
    activated() {
      //一进入组件就滚动到离开时保存的位置
      this.$refs.scroll && this.$refs.scroll.scrollTo(0, this.saveY, 10);
      //refresh
      // this.$refs.scroll && this.$refs.scroll.refresh();
    },
    deactivated() {
      //保存离开时的位置
      this.saveY = this.$refs.scroll.getScrollY();
      // 2、取消全局事件监听
      this.$bus.$off('itemImgLoad', this.itemImgListener);
    },
    methods: {
      /**
       * 事件相应相关方法
       */
      //tabControl的点击，mixin里的tabClick发生点击事件并调用此方法
      _tabClick(index) {
        this.$refs.tabControl1.currentIndex = index;
        this.$refs.tabControl2.currentIndex = index;
      },
      //监听点击了那个分类栏目
      selectItem(index) {
        // console.log(index);
        this._getSubCategories(index);
      },
      showiid(v) {
        this.iid=v
        //console.log(this.currentIndex);
        this._getSubCategories(this.currentIndex);

      },
      //监听滚动的位置（使用scroll组件传过来的事件）
      contentScroll(position) {
        //监听BackTop是否显示
        this.isShowBackTop = position.y < TOP_DISTANCE;
        //判断tabControl是否需要吸顶
        this.isTabFixed = this.tabOffsetTop < (-position.y);
      },
      /**
       * 网络请求额相关法
       */
      //1、拿到分类数据
      _getCategory() {
        getCategory().then(res => {
          // console.log(res.data);
          //保存分类数据
          this.categories = res.data.category.list;
          //初始化每个类别的子数据
          for(let i = 0; i < this.categories.length; i++) {
            this.categoryData[i] = {
              subcategories: {},
              categoryDetail: {
                'pop': [],
                'new': [],
                'sell': []
              }
            }
          }
          //请求第一个分类的数据
          this._getSubCategories(0);
        })
      },
      //根据分类栏的下标去请求相应的数据
      _getSubCategories(index) {
        this.currentIndex = index;
        // console.log(this.categories);
        const mailKey = this.categories[index].maitKey;
        getSubcategory(mailKey).then(res => {
          this.categoryData[index].subcategories = res.data;
          this.categoryData = {...this.categoryData};
          this._getCategoryDetail(POP)
          this._getCategoryDetail(SELL)
          this._getCategoryDetail(NEW)
        })
      },
      //3、根据类型请求相应的商品数据（pop,new,sell）
      _getCategoryDetail(type) {
        //获取请求的minWallkey
        const miniWallkeys =this.categories.length==0?this.iid: this.categories[this.currentIndex].miniWallkey;
        //const miniWallkey =this.categories[this.currentIndex].miniWallkey;
        //发送请求,传入miniWallkey和type
        getCategoryDetail(miniWallkeys, type).then(res => {
          //将获取的数据保存下来
          this.categoryData[this.currentIndex].categoryDetail[type] = res;
          this.categoryData = {...this.categoryData};
        })
      }
    }
  }
</script>

<style scoped>
  .category {
    height: 100%;
  }
  .nav-bar {
    position: relative;
    background-color: var(--color-tint);
    color: #fff;
    font-weight: 700;
    z-index: 3;
    text-align: center;
  }
  .content {
    position: absolute;
    left: 0;
    right: 0;
    top: 44px;
    bottom: 49px;
    display: flex;
  }
  #tab-content {
    overflow: hidden;
    height: calc(100vh - 49px - 44px);
    flex: 1;
  }
  .tab-control2 {
    position: absolute;
    width: calc(100% - 60px);
    z-index: 3;
    right: 0;
  }
</style>
