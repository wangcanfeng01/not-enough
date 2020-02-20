# 前言
在自建网站中我使用了pegdown这个jar包解析md文档，然后返回给前端做展示用，但是没有导航很难受，所以制作了一个简单的导航生成工具。索引解析速度还行，不会导致页面卡顿，各个锚点定位也很准确，就是整个风格有点丑，各位童鞋使用的时候可以修改一下
``` xml
<template>
  <div id="test">
    <el-col :span="4" :offset="1" style="margin-top: 20px;position: fixed">
      <div ref="nav-info" class="test"></div>
    </el-col>
    <el-col :span="12" :offset="6">
      <div ref="text_info">
        <h1>标题1</h1>
        <p>wwwww</p>
        <h2>2级</h2>
        <p>测试文本内容</p>
        <p>dsadasdas</p>
        <p>dsadasdas</p>
        <p>dsadasdas</p>
        <p>dsadasdas</p>
        <h3>3级</h3>
        <img src="../assets/home/blog.jpg" style="width: 350px;height: 350px;margin-left: 5em"/>
        <h3>3级</h3>
        <img src="../assets/home/blog.jpg" style="width: 350px;height: 350px;margin-left: 5em"/>
        <h1>标题2</h1>
        <img src="../assets/home/blog.jpg" style="width: 350px;height: 350px;margin-left: 5em"/>
        <h2>2级</h2>
      </div>
    </el-col>
    <a href="#wcf_blog_10">sssss</a>
  </div>
</template>

<script>
export default {
  name: 'test',
  data () {
    return {
      navVisible: false,
      indexes: []
    }
  },
  methods: {
    getNav () {
      let text = this.$refs['text_info']
      this.$refs['nav-info'].innerHTML = text.innerHTML
      let nodes = this.$refs['nav-info'].children
      if (nodes.length) {
        for (let i = 0; i < nodes.length; i++) {
          this.createNav(nodes[i], i)
        }
      }
      // 插入导航锚点
      this.insertIndexElement(text)
      // 创建导航头
      this.createTitle(nodes[0])
    },
    // 制作导航函数，整个方法的核心
    createNav (node, i) {
      // 设置正则表达式
      let reg = /^H[1-6]{1}$/
      if (!reg.exec(node.tagName)) {
        // 如果不是标题信息，不展示
        node.style.display = 'none'
      } else {
        // 存入索引数组，等会用于生成锚点
        this.indexes.push(i)
        // 生成导航元素
        let ele = document.createElement('a')
        ele.innerText = node.innerText
        // 删除原来标题中的信息
        node.innerText = ''
        ele.href = '#wcf_blog_' + i
        // 插入导航元素
        node.appendChild(ele)
      }
    },
    // 插入锚点的函数
    insertIndexElement (info) {
      let len = this.indexes.length
      if (len > 0) {
        // 从后往前插入，从前往后插入会导致index位移
        for (let i = len - 1; i >= 0; i--) {
          let ele = document.createElement('a')
          ele.setAttribute('id', 'wcf_blog_' + this.indexes[i])
          info.insertBefore(ele, info.children[this.indexes[i]])
        }
      }
    },
    // 生成导航头
    createTitle (first) {
      let title = document.createElement('h1')
      title.innerText = '导航'
      title.style.fontSize = '2em'
      title.style.fontFamily = 'Arial,sans-serif'
      title.style.paddingBottom = '15px'
      title.style.borderBottom = 'solid black 1px'
      title.style.marginRight = '20px'
      this.$refs['nav-info'].insertBefore(title, first)
    }
  },
  mounted () {
    this.getNav()
  }
}
</script>

<style scoped>
  .test {
    border: solid #409EFF 1px;
    padding-left: 20px;
    border-radius: 15px;
    min-height: 750px;
    color: #409EFF
  }

  .test h1 {
    font-size: 1.7em;
  }

  .test h2 {
    margin-left: 20px;
  }

  .test h3 {
    margin-left: 35px;
  }

  .test h4 {
    margin-left: 45px;
  }

  .test h5 {
    margin-left: 55px;
  }
</style>

```
原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)