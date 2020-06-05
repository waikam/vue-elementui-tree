<template>
  <div id="app">
    <el-button type="primary" @click="showDialog">点击弹出弹框</el-button>
    <div v-show="chapterIds.length === 0">请点击按钮进行选择</div>
    <div v-show="chapterIds.length>0" class="list">
      <div v-for="(item, index) in chapterLists" :key="index">{{index+1}} - {{item}}</div>
    </div>
    <myDialog :showDialogState.sync="showDialogState">
      <el-scrollbar style="height: 500px;">
        <div v-infinite-scroll="load" class="tree-container" infinite-scroll-immediate>
          <el-tree
            :data="chapterTreeData"
            show-checkbox
            default-expand-all
            highlight-current
            check-strictly
            node-key="id"
            ref="chapterTree"
            :props="{ children: 'childrenList', label: 'name' }"
            @check="handleCheckChange"
          ></el-tree>
        </div>
      </el-scrollbar>
      <div class="tree-footer">
        <el-button @click="cancelTree">取消</el-button>
        <el-button type="primary" @click="configTree">确定</el-button>
      </div>
    </myDialog>
  </div>
</template>

<script>
import $http from './router/http'
import myDialog from '@/components/my-dialog.vue'
export default {
  name: 'app',
  components: {
    myDialog
  },
  data() {
    return {
      showDialogState: false,
      pageData: {
        pageSize: 20,
        pageNum: 1,
        pageTotal: 0
      },
      lastPages: false,
      chapterTreeData: [], // 树组件的数据
      choiceTreeData: [], // 已选中的数据
      chapterIds: [], // 最终可以传递给父组件的已选中的数据
      chapterLists: []
    }
  },
  created() {
  },
  methods: {
    // 树组件触底加载
    load() {
      // 不是最后一页就获取数据
      if (this.pageData.pageNum < this.pageData.pageTotal) {
        this.pageData.pageNum = this.pageData.pageNum + 1
        this.lastPages = false
        this.getChapterTreeData()
      } else { // 是最后一页就不再获取数据
        this.lastPages = true
      }
    },
    // 接口方法-获取单元树数据
    async getChapterTreeData() {
      let data = { ...this.pageData, bookVersionId: 'cfa209ddd5e158a26753b30c58d7eb54' }
      let res = await $http({
        url: '/course/chapter/treeList', // 改成自己的接口地址 
        method: 'get',
        datatype: 'application/json',
        params: { ...data },
      })
      if (this.lastPages) {
        // 如果是最后一页或者是一开始就不合并数据
        this.chapterTreeData = res.data.records 
      } else {
        // 如果不是最后一页就合并数组
        this.chapterTreeData = this.chapterTreeData.concat(res.data.records) 
      }
      this.pageData.pageTotal = res.data.pages
    },
    // 点击按钮显示弹框
    showDialog() {
      this.showDialogState = true
      this.lastPages = true
      // 返回到第一页，避免之前打开过显示底部的数据
      this.pageData = {
        pageSize: 20,
        pageNum: 1,
        pageTotal: 0
      }
      this.getChapterTreeData(this.pageData)
      // 如果有选中的话，就默认选中已有数据
      if (this.chapterIds.length > 0) {
        // 改变选中状态  setCheckedKeys 是tree组件提供的方法
        this.$nextTick(function () {
          this.$refs.chapterTree.setCheckedKeys(this.chapterIds.split(','))
        })
      }
    },
    // 复选框被点击
    handleCheckChange(data) {
      // choiceState表示当前数据是否是选中的状态
      let choiceState = true
      // 判断已选中的数据中是否包含当前被点击的数据
      for (let i in this.choiceTreeData) {
        choiceState = this.choiceTreeData[i] === data.id ? false : true
      }
      // 已选数据里面不包含当前被点击的数据或者没有已选数据，就选中该数据
      if ((this.choiceTreeData.length > 0 && choiceState) || this.choiceTreeData.length <= 0) {
        /** 这里由于后端同事返回的数据里面会将子节点的父节点的id同时返回给我，
         * 所以我很容易的获取到到了当前选中的节点的父节点，然后稍微处理一下就能获取到父节点的id了，
         * 然后通过组件提供的setCheckedKeys方法设置默认选中的节点就好了，如果后端同事没有返回的话，
         * 就要自己讲数据循环获取到当前父节点的id，会比较麻烦 */
        let dataIdArr = []
        dataIdArr.push(data.parentCodes, data.id)
        let strDataId = dataIdArr.join(',')
        this.choiceTreeData = strDataId.split(',')
        this.choiceTreeData.splice(0, 1)
        this.$refs.chapterTree.setCheckedKeys(this.choiceTreeData)
        // 获取选中的数据的名称，方便展示选中了什么
        let choiceList = this.$refs.chapterTree.getCheckedNodes()
        let choiceListName = []
        for(let i in choiceList) {
          choiceListName.push(choiceList[i].name)
        }
        this.chapterLists = choiceListName
      } else {  // 当前选中的数据跟已选数据相同，就取消选中状态
        this.choiceTreeData = []
        this.$refs.chapterTree.setCheckedKeys(this.choiceTreeData)
      }
    },
    // 点击树弹框的确定按钮
    configTree() {
      // 如果没有选择的话就提示要选择
      if (this.choiceTreeData.length <= 0) return this.$common.tipsPop('请选择单元', 'error')
      this.showDialogState = false
      // 这里this.chapterIds就获取到了当前节点的id和当前节点的父节点id了
      this.chapterIds = this.choiceTreeData.join(',')
    },
    // 点击树弹框的取消按钮
    cancelTree() {
      // 如果有已经有数据的话 就不影响原有的数据，
      if (this.chapterIds.length > 0) {
        this.choiceTreeData = this.chapterIds.split(',')
        this.$nextTick(function () {
          this.$refs.chapterTree.setCheckedKeys(this.chapterIds.split(','))
        })
      } else {
        // 如果之前没有选中数据的话，那么就将默认选中设置为空
        this.choiceTreeData = []
        this.$nextTick(function () {
          this.$refs.chapterTree.setCheckedKeys([])
        })
      }
      this.showDialogState = false
    },
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
.list {
  width: 30%;
  margin: 10px auto;
  text-align: left;
  line-height: 28px;
}
.el-tree-node__content {
  height: 40px !important;
  line-height: 40px !important;
  border-bottom: 1px solid #e0e0e0;
}

.el-tree > .el-tree-node {
  background: #f7f7f7;
  font-weight: bold;
  font-size: 15px;
}
.el-tree-node__content {
  border-bottom: 1px solid #eaeaea;
}
.el-tree > .el-tree-node > .el-tree-node__children {
  font-weight: normal;
  background: #fff;
  font-size: 13px;
}
.el-tree-node__content {
  border-bottom: 1px solid #f1f1f1;
}
.el-tree-node__content:hover {
  background: #f3fcff;
}
</style>
