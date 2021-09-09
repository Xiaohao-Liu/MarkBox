<template>
  <div id="app_" :class="'main'+(darkmode?' dark_mode':'')">
    <div id="main">
      <el-container v-loading="loading">
        <el-header id="header">
          <div 
          :class="'header_item '+(item.enable?'enable':'disable')" 
          :style="'width:'+item.width+'px;'+item.style" 
          v-for="item in tool_bar_list" :key="item.name"
          v-show="item.show"
          >
            <div class="header_btn" v-on:click="(item.enable&&item.type!='btn'&&item.click())">
              <i :class="item.icon"/> {{item.title}}
              <el-switch v-if="item.type=='btn'"
                v-model="item.enable"
                active-color="#13ce66"
                inactive-color="#ff4949"
                @change="item.click()">
              </el-switch>
            </div>
            <div class="header_name">{{item.name}}</div>
          </div>
          <div class="header_item" style="width:100px;"
          >
            <div class="header_btn" >
              {{count}}
            </div>
            <div class="header_name">字数统计</div>
          </div>
        </el-header>
        <el-main style="height:calc(100vh - 60px);padding:0px;position: relative;top:60px;text-align:left;" id="md_container" :class="full_screen?'full_screen vditor':'vditor'"></el-main>
        <el-button class="full_screen_button" icon="el-icon-full-screen" v-if="full_screen" @click="full_screen=false"></el-button>
        <div class="big_board" v-show="pushed_pic_config.show" >
          <div class="behind_" @click="pushed_pic_config.show=false"></div>
          <el-card class="edit_board" v-loading="pushed_pic_config.loading">
              <img id="pushed_image" :src="pushed_pic_config.base64||pushed_pic_config.url" style="width:100%;"/>  
                <p v-show="pushed_pic_config.base64=='' && pushed_pic_config.url==''  && !pushed_pic_config.pushed" style="text-align:center;">NO IMAGE!</p>   
                <!-- <el-input placeholder="照片名称" v-model="pushed_pic_config.name" v-show="!pushed_pic_config.pushed && pushed_pic_config.url.length==0">
                    <el-button slot="append" icon="el-icon-upload" v-on:click="push_a_pic"></el-button>
                </el-input> -->
                <el-input placeholder="照片链接" v-model="pushed_pic_config.url" style="margin-top:10px;"></el-input>
                <el-input placeholder="照片链接" v-model="pushed_pic_config.name" style="margin-top:10px;"></el-input>
                <div class="btn green" v-if="pushed_pic_config.url.length>0&&pushed_pic_config.name.length>0" v-on:click="__upload_pic_to_itee_file">上传图片</div>
          </el-card>
        </div>
        <div class="big_board" v-show="md_head.show" >
          <div class="behind_" @click="md_head.show=false"></div>
          <el-card v-show="md_head.show" class="edit_board" v-loading="md_head.loading">  
              <el-input placeholder="题目" v-model="md_head.title" style="margin-top:10px;"></el-input>
              <el-input placeholder="备注（一般为作者名）" v-model="md_head.remark" style="margin-top:10px;"></el-input>
              <div class="btn green" v-if="md_head.title.length>0" v-on:click="add_head()">添加到文档中</div>
          </el-card>
        </div>
        <div class="first_load_cover" v-if="first_load">
          <div class="first_load_board">
            <div v-if="read_mode">
              {{cover_msg}}
            </div>
            <div v-else>
              Choose a Operation:
              <div class="btn green iconfont icon-new" v-on:click="__init_a_blank_project"> Create</div>
              <div class="btn green iconfont icon-storage" v-on:click="__init_an_online_project" v-if="gitee_enable"> Open</div>
            </div>
          </div>
        </div>
        <div class="big_board" v-show="file_config.show" >
          <div class="behind_" @click="file_config.show=false"></div>
            <div class="config_board file_config_board" v-if="!read_mode&&file_config.show" v-loading="file_config.loading">
              <div class="board_title">{{file_config.title}}</div>
              <div style="display:flex;">
                <div class="btn blue iconfont icon-refresh" v-on:click="__get_gitee_files"></div>
                <div class="btn blue iconfont icon-new" v-on:click="__add_file_group"></div>
                <div class="btn green iconfont icon-upload" v-on:click="__upload_file_group"></div>
              </div>
              <div class="group_item" v-for="group,idx in file_config.groups" :key="group.name">
                <p style="font-size:1.2rem;font-weight:bold;line-height:20px;margin:5px 0px;">{{group.name}}</p>
                <div class="btn red" style="width:calc(100% - 30px);" v-show="group.list.length<=0" v-on:click="__delete_file_group(idx)">Delete Group</div>
                <draggable
                  class="list-group"
                  :list="group.list"
                  group="files"
                >
                  <div class="file_item list-group-item" v-for="item in group.list" :key="item.path">
                    <div  style="width:calc(100% - 30px);float:left;white-space: nowrap;overflow: hidden;text-overflow: ellipsis;"  v-on:click="__load_gitee_file(item.path,item.sha);">
                      <i class="el-icon-document"/>{{item.path}}
                    </div>
                  </div>
                </draggable>
              </div>
            </div>
        </div>
      </el-container>
    </div>
  </div>
</template>

<script>
const systemDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
import draggable from "vuedraggable";

import {getGiteeInfo, GiteeAPI, removeToken} from "../api";
import "../assets/iconfont/iconfont.css"

import Vditor from 'vditor'
import "vditor/src/assets/scss/index.scss"

export default {
  name: 'HelloWorld',
  props: {
    // msg: String
  },
  components: {draggable},
  data:function(){
    return {
      gitee_info: {enable:false, username:"",repos:"",token:""},
      editor: null,
      count:0,
      show_menu: false,
      server_mode:false,
      full_screen: false,
      read_mode:false,
      cover_msg:"加载中...",
      darkmode:systemDark,
      loading:false,
      first_load:true,
      gitee_enable: false,
      file_name:"untitled",
      file_sha:"null",
      file_type:"mdb",
      online_file:false,
      selected_node:null,
      selected_edge:null,
      predefineColors:["#eeeeee","#ffffff","#000000","#b39ddb","#f44336","#009688","#0d47a1"],
      request_lock:false,
      zoom:0,
      md_head:{
        show:false,
        title:"",
        remark:"",
        loading:false
      },
      tool_bar_list:[
        {name:"Xiaohao Liu",
          width:100,
          style:"margin-right: 20px;padding-right: 20px;",
          enable:true,
          show:true,
          icon:"",
          title:"MarkBox",
          click:()=>{this.index()},
        },
        {name:"文件名",
          width:200,
          style:"margin-right: 20px;padding-right: 20px;",
          enable:true,
          show:true,
          icon:"",
          title:"null",
          click:()=>{this.__file_share_link()},
        },
        {
          name:"图片",
          width:40,
          style:"",
          enable:true,
          show:true,
          icon:"iconfont icon-pic",
          title:"",
          click:()=>{this.__tool_add_pic()},
        },
        {
          name:"上传",
          width:40,
          style:"",
          enable:true,
          show:true,
          icon:"iconfont icon-upload",
          title:"",
          click:()=>{this.__upload_to_gitee()},
        },
        {
          name:"深色模式",
          width:100,
          style:"",
          enable:this.darkmode,
          show:true,
          type:'btn',
          icon:"",
          title:"",
          click:()=>{this.darkmode = !this.darkmode;this.tool_bar_list[this.tool_map['darkmode']].enable=this.darkmode;},
        },
        {
          name:"添加头",
          width:40,
          style:"",
          enable:true,
          show:true,
          icon:"el-icon-menu",
          title:"",
          click:()=>{this.md_head.show=!this.md_head.show},
        },
        // {
        //   name:"全屏模式",
        //   width:100,
        //   style:"",
        //   enable:true,
        //   show:true,
        //   icon:"el-icon-full-screen",
        //   title:"",
        //   click:()=>{this.full_screen=true},
        // },
        {
          name:"登出",
          width:60,
          style:"float:right",
          enable:true,
          show:true,
          icon:"el-icon-coffee-cup",
          title:"",
          click:()=>{removeToken();this.$router.push("/login")},
        },
      ],
      tool_map:{
        "file":1,
        "pic":2,
        "upload":3,
        "darkmode":4,
      },
      read_mode_bar:new Set([0,1,4]),
      pushed_pic_config:{
        show:false,
        name:"",
        base64:"",
        pushed:false,
        raw:"",
        loading:false,
        url:"",
      },
      file_config:{
        show:false,
        title:'Files',
        label:"",
        loading:false,
        groups:[
          
        ],
        files:new Set()
      },
      markdown_mode:false
    }
  },
  watch:{
    darkmode: function(pre, now){
      if(!now)this.editor.setTheme('dark','dark');
      else this.editor.setTheme('classic','light');
    }
  },
  mounted:function(){
    getGiteeInfo().then(res=>{
      this.server_mode = true;
      this.gitee_info = res;
      this.gitee_enable = this.gitee_info.enable;
    }).catch(err=>{
      console.log(err)
    }).finally(()=>{
      this.__init_read_mode();
      this.__init_markdown();

      this.__add_events();
      if(this.read_mode){this.__load_online_file()}
    })
    
  },
  methods:{ 
    index:function(){
      if(!this.read_mode&&this.gitee_enable){
        this.file_config.show = !this.file_config.show;
        if(this.file_config.show==true){
          if(this.can_request()){this.__get_gitee_files();this.send_a_request()}
        }
      }
      else{
        alert("MarkBox\nDesined by Xiaohao Liu.\nGithub: WYKXLDZ");
        }
    },
    __init_read_mode:function(){
      if(!this.gitee_enable)this.read_mode=true;
      const location_parames = {}
      window.location.search.substring(1).split("&").forEach(ele=>{
        let i = ele.split("=");
        location_parames[i[0]] = i[1]
      })
      try{
        let gitee_info={
          enable:true,
          username:location_parames['u'],
          repos: location_parames['r'],
          token:"",
        }
        
        this.file_sha = location_parames['s']
        this.file_name = location_parames['n'].split(".mdb")[0]
        
        this.read_mode=true;
        // console.log(this.gitee_info)
        this.gAPI = new GiteeAPI(gitee_info);
        this.gitee_info = gitee_info
        this.tool_bar_list.forEach((ele,idx)=>{
          if(this.read_mode_bar.has(idx))return;
          ele.show=false;
        })
      }catch{
        if(!this.read_mode){this.gAPI = new GiteeAPI(this.gitee_info);return;}
        else{
          this.cover_msg="参数错误！"
        }
        return;
      }
    },
    __load_online_file:function(){
      if(!this.can_request()){return;}
      this.send_a_request();
      this.loading=true;
      this.gAPI.get_file_by_sha(this.file_sha).then(res=>{
        this.editor.setValue(this.__decode(res.data.content))
        this.tool_bar_list[this.tool_map['file']].title=this.file_name;
        this.loading=false;
        this.online_file=true;
        this.first_load=false;
        document.getElementsByTagName('title')[0].innerText = this.file_name;
      }).catch((err)=>{
        this.cover_msg="Wrong\n" + err
        console.log("wrong:", err)
        this.loading=false;
      })
      
    },
    __init_markdown:function(){
        
        this.editor = new Vditor("md_container",
        {
          height: 360,
          counter:{
            enable:true,
            after:(nu)=>{this.count = nu}
          },
          preview:{
            hljs:{
              lineNumber:true
            },
            markdown:{
              toc:true
            }
          },
          outline:{
            enable:true
          },
          toolbarConfig: {
            hide: false,
          },
          toolbar:[],
          cache: {
            enable: false,
          },
          theme:"classic",
          after: () => {
            // darkmode
            this.tool_bar_list[this.tool_map['darkmode']].enable=this.darkmode;
            if(this.darkmode)this.editor.setTheme('dark','dark');
          },
        })

    },
    add_head:function(){
      this.editor.setValue( this.editor.html2md(
        '<pre ><p class="head_title" style="text-align:center;font-size:2.2rem;font-weight:bold;border-bottom:4px double black;"> '
        +this.md_head.title+
        ' </p><p style="text-align:center;font-size:1.2rem;font-weight:bold;"> '
        +this.md_head.remark+ 
        '</p> </pre>')
        + this.editor.getValue()
        )
        this.md_head.show=false;
    },
    send_a_request:function(){
      this.request_lock=true;
      setTimeout(()=>{
        this.request_lock=false;
      },2000)
    },
    can_request:function(){
      if(this.request_lock==true){
        this.$notify({
            title: '请求速率过快',
            message: "2s后重试"
        });
        return false;
      }
      return true;
    },
    __init_an_online_project:function(){
      this.file_config.show=true;
      this.__get_gitee_files();
      this.first_load=false;
      this.online_file=true;
    },
    __init_a_blank_project:function(){
      // this.graph.fromJSON({})
      // this.graph.centerContent();
      this.tool_bar_list[this.tool_map['file']].title=this.file_name;
      this.file_name = "untitled";
      this.tool_bar_list[this.tool_map['file']].title=this.file_name;
      this.first_load=false;
      this.online_file=false;
      document.getElementsByTagName('title')[0].innerText = "new file";
    },
    __enable:function(name){
      this.tool_bar_list[this.tool_map[name]].enable=true;
    },
    __disable:function(name){
      this.tool_bar_list[this.tool_map[name]].enable=false;
    },

    __add_events:function(){
  
      this.__add_paste_event()
    },
    __add_keyboard_events:function(){
      this.graph.bindKey('enter',()=>{this.__tool_add_sibling()})
      this.graph.bindKey('ctrl+enter',()=>{this.__tool_add_child()})
      this.graph.bindKey('backspace',()=>{this.__del_node()})
      this.graph.bindKey('ctrl+c', () => {
        const cells = this.graph.getSelectedCells()
        if (cells.length) {
          const new_cells = []
          cells.forEach(cell=>{
            if(cell.store.data.idx!=0){
              new_cells.push(cell);
            }
          })
          this.graph.copy(new_cells)
        }
        return false
      })
      this.graph.bindKey('ctrl+v', () => {
          if (!this.graph.isClipboardEmpty()) {
            const cells = this.graph.paste({ offset: 32 })
            this.graph.cleanSelection()
            this.graph.select(cells)
          }
          return false
        })
      this.graph.bindKey('ctrl+z', () => {
        this.history.canUndo()?this.graph.undo():false;
      })
      this.graph.bindKey('ctrl+s', () => {
        this.__upload_to_gitee();
      })
      this.graph.bindKey('ctrl+n', () => {
        this.__init_a_blank_project();
      })

    },
    __add_paste_event:function(){
        const stopwords = ",.'\":; "
        document.addEventListener('paste', (event) =>{
            if(!this.pushed_pic_config.show){
                return;
            }
            
            var items = event.clipboardData && event.clipboardData.items;
            var file = null;
            var reader  = new FileReader();
            if (items && items.length) {
                // 检索剪切板items
                for (var i = 0; i < items.length; i++) {
                    if (items[i].type.indexOf('image') !== -1) {
                        file = items[i].getAsFile();
                        break;
                    }
                }
            }
            var name = this.file_name.substring(0,15).toLowerCase();
            for(var w =0;w<stopwords.length;w++){
                name = name.split(stopwords[w]).join("_")
            }
            name= name+"_1.png"
            reader.onload = (event)=>{
                this.pushed_pic_config.base64 = event.target.result;
                this.pushed_pic_config.url = this.pushed_pic_config.base64;
                this.pushed_pic_config.name = name;
                this.pushed_pic_config.pushed = false;
            }
            if(file){
                reader.readAsDataURL(file)
            }
        });
    },

    __tool_add_pic:function(){
      this.pushed_pic_config.show = !this.pushed_pic_config.show;
    },
    __file_share_link:function(){
      if(this.online_file){
        const file_link = "http://wykxldz.gitee.io/markbox/?u="+this.gitee_info.username+"&r="+this.gitee_info.repos+"&s="+this.file_sha+"&n="+this.file_name+".mdb"
        this.$alert('', '分享阅读链接', {
          showInput:true,
          inputValue:file_link
        });
        return;
      }
      this.$prompt('请输入文件名称', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          inputPattern: /^([0-9a-zA-z]{1})([\w]*)$/g,
          inputPlaceholder:this.file_name,
          inputErrorMessage: '文件格式不正确'
        }).then(({ value }) => {
          if(this.file_config.files.has(value)){
            this.$message({
              type: 'warning',
              message: '已存在该文件名'
            });
            return;
          }
          this.file_name=value;
          this.tool_bar_list[this.tool_map['file']].title=this.file_name;
          this.$message({
            type: 'success',
            message: '修改成功'
          });
        }).catch(() => {
          this.$message({
            type: 'info',
            message: '取消输入'
          });       
        });
      
    },
    __get_gitee_files:function(){
      if(this.gitee_info.username == "" || this.gitee_info.repos == "" || this.gitee_info.token == "" ){
          this.$notify({
              title: ' 失败',
              message: "Gitee 信息有误"
          });
          return;
      }
      this.file_config.loading=true;
      this.gAPI.get_files().then(res=>{
        let tree = res.data.tree;
        this.file_config.groups.splice(0,this.file_config.groups.length);
        
        for(let i = 0; i < tree.length;i++){
          if(tree[i].path=="MarkBox_file_groups"){
            this.gAPI.get_file_by_sha(tree[i].sha).then(res=>{
              this.file_config.groups = JSON.parse(this.__decode(res.data.content))
            }).catch(()=>{
              this.file_config.groups.push({
                name: "files",
                list: []
              })
            }).finally(()=>{
              let s = new Set();
              let s_ = new Set();
              let s_map = {}
              for(let i = 1; i< this.file_config.groups.length; i++){
                this.file_config.groups[i].list.forEach((ele)=>{
                  s.add(ele.path.split('.'+this.file_type)[0])
                })
              }
              this.file_config.groups[0].list.splice(0,this.file_config.groups[0].list.length)
              for(let i = 0; i < tree.length;i++){
                let d = tree[i].path.split(".")
                if(d[d.length-1]==this.file_type ){
                  let name = tree[i].path.split('.'+this.file_type)[0]
                  this.file_config.files.add(name)
                  if(s.has(name)){
                    s_.add(name);s_map[name] = tree[i]
                  }
                  else this.file_config.groups[0].list.push(tree[i]);
                }
              }
              // delete the files that not exist
              function difference(setA, setB) {
                  let _difference = new Set(setA);
                  for (let elem of setB) {
                      _difference.delete(elem);
                  }
                  return _difference;
              }
              let d_ = difference(s,s_)
              for(let i = 1; i< this.file_config.groups.length; i++){
                this.file_config.groups[i].list.forEach((ele,idx)=>{
                  let name = ele.path.split('.'+this.file_type)[0]
                  if(d_.has(name)){
                    this.file_config.groups[i].list.splice(idx,idx+1)
                  }
                  if(s_.has(name)){
                    for(let j in ele){ele[j] = s_map[name][j]}
                  }
                })
              }
              this.file_config.loading=false;
            })
            break;
          }
        }
        this.file_config.groups.push({name:"Files",list:[]})
        for(let i = 0; i < tree.length;i++){
          let d = tree[i].path.split(".")
          if(d[d.length-1]==this.file_type ){
              this.file_config.groups[0].list.push(tree[i]);
          }
        }
        this.file_config.loading=false
        
      }).catch(()=>{
        this.file_config.loading=false;
      })
      
    },
    __load_gitee_file:function(name,sha){
      if(this.gitee_info.username == "" || this.gitee_info.repos == "" || this.gitee_info.token == "" ){
          this.$notify({
              title: ' 失败',
              message: "Gitee 信息有误"
          });
          return;
      }
      if(!this.can_request()){return;}
      this.send_a_request();
      this.loading=true;
      this.file_config.show=false;
      this.gAPI.get_file_by_sha(sha).then(res=>{
        this.mddata = this.__decode(res.data.content)
        
        this.editor.setValue(this.mddata)
        this.file_name = name.split(".mdb")[0];
        this.file_sha = sha;
        this.tool_bar_list[this.tool_map['file']].title=this.file_name;
        this.loading=false;
        this.online_file=true;
        document.getElementsByTagName('title')[0].innerText = this.file_name;
      }).catch(()=>{
        this.loading=false;
      })
      
    },
    __upload_pic_to_itee_file:async function(){ 
        let content = this.pushed_pic_config.base64.substring(22);
        let name = this.pushed_pic_config.name;
        return new Promise((resolve,reject)=>{
          if(this.gitee_info.username == "" || this.gitee_info.repos == "" || this.gitee_info.token == "" ){
              this.$notify({
                  title: ' 失败',
                  message: "Gitee 信息有误"
              });
              reject(false)
              return false;
          }
        this.pushed_pic_config.loading = true;
          if(!this.can_request()){reject(false);return false;}
          this.send_a_request();
          this.gAPI.get_file_by_path(name).then(res=>{
            if(res.data.length == 0){
              const data = {
                "content":content
              }
              this.gAPI.new_file(name,data).then((res)=>{
                this.$notify({
                    title: '成功',
                    message: name+" 上传成功！"
                });
                this.pushed_pic_config.url = res.data.content.download_url;
                this.pushed_pic_config.loading = false;
                resolve(res.data)
              }).catch((err)=>{
                this.$notify({
                    title: ' 失败',
                    message: err.data.responseJSON.message
                });
                this.pushed_pic_config.loading = false;
                reject(err.data)
              })
            }
            else{
              this.$alert('是否替换图片'+name, '替换', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
              }).then(() => {
                const data ={
                  "content":content,
                  "sha":res.data.sha
                }
                this.gAPI.update_file(name,data).then((res)=>{
                  this.$notify({
                      title: '成功',
                      message: name+" 替换成功！"
                  });
                  this.pushed_pic_config.url = res.data.content.download_url;
                  this.pushed_pic_config.loading = false;
                  resolve(res.data)
                }).catch((err)=>{
                  this.$notify({
                    title: ' 失败',
                    message: err.data.responseJSON.message
                  });
                  this.pushed_pic_config.loading = false;
                  reject(err.data)
                })
              }).catch((err)=>{
                this.$notify({
                  title: '取消替换',
                  message: ''
                });
                this.pushed_pic_config.loading = false;
                reject(err.data)
              })
            }
          }).catch(()=>{
            this.$notify({
                title: ' 失败',
                message: '请求出错'
            });
            this.pushed_pic_config.loading = false;
          })
        })
    },
    __upload_gitee_file:async function(name,content){ 
        return new Promise((resolve,reject)=>{
          if(this.gitee_info.username == "" || this.gitee_info.repos == "" || this.gitee_info.token == "" ){
              this.$notify({
                  title: ' 失败',
                  message: "Gitee 信息有误"
              });
              reject(false)
              return false;
          }
          if(!this.can_request()){reject(false);return false;}
          this.send_a_request();
          this.gAPI.get_file_by_path(name).then(res=>{
            if(res.data.length == 0){
              const data = {
                "content":content
              }
              this.gAPI.new_file(name,data).then((res)=>{
                this.$notify({
                    title: '成功',
                    message: name+" 上传成功！"
                });
                this.file_sha = res.data.content.sha
                this.file_name = res.data.content.name.split(".mdb")[0]
                resolve(res.data)
              }).catch((err)=>{
                this.$notify({
                    title: ' 失败',
                    message: err.data.responseJSON.message
                });
                reject(err.data)
              })
            }
            else{
              this.$alert('是否替换'+name, '替换', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
              }).then(() => {
                const data ={
                  "content":content,
                  "sha":res.data.sha
                }
                this.gAPI.update_file(name,data).then((res)=>{
                  this.$notify({
                      title: '成功',
                      message: name+" 替换成功！"
                  });
                  console.log(res.data)
                  this.file_sha = res.data.content.sha
                  this.file_name = res.data.content.name.split(".mdb")[0]
                  resolve(res.data)
                }).catch((err)=>{
                  this.$notify({
                    title: ' 失败',
                    message: err.data.responseJSON.message
                  });
                  reject(err.data)
                })
              }).catch((err)=>{
                this.$notify({
                  title: '取消替换',
                  message: ''
                });
                reject(err.data)
              })
            }
          }).catch(()=>{
            this.$notify({
                title: ' 失败',
                message: '请求出错'
            });
          })
        })
        
    },
    __upload_to_gitee:function(){
      const data = this.editor.getValue()
      const content = this.__encode(data);
      const name = this.file_name+'.'+this.file_type;
      this.loading=true;
      this.__upload_gitee_file(name,content).then(()=>{
        this.loading=false;
      }).catch(()=>{
        this.loading=false;
      })
    },
    __upload_file_group:function(){
      let content = this.__encode(JSON.stringify(this.file_config.groups));
        let name = "MarkBox_file_groups";
        return new Promise((resolve,reject)=>{
          if(this.gitee_info.username == "" || this.gitee_info.repos == "" || this.gitee_info.token == "" ){
              this.$notify({
                  title: ' 失败',
                  message: "Gitee 信息有误"
              });
              reject(false)
              return false;
          }
        this.file_config.loading = true;
          if(!this.can_request()){reject(false);return false;}
          this.send_a_request();
          this.gAPI.get_file_by_path(name).then(res=>{
            if(res.data.length == 0){
              const data = {
                "content":content
              }
              this.gAPI.new_file(name,data).then((res)=>{
                this.$notify({
                    title: '成功',
                    message: name+" 上传成功！"
                });
                this.file_config.loading = false;
                resolve(res.data)
              }).catch((err)=>{
                this.$notify({
                    title: ' 失败',
                    message: err.data.responseJSON.message
                });
                this.file_config.loading = false;
                reject(err.data)
              })
            }
            else{
                const data ={
                  "content":content,
                  "sha":res.data.sha
                }
                this.gAPI.update_file(name,data).then((res)=>{
                  this.$notify({
                      title: '成功',
                      message: name+" 替换成功！"
                  });
                  this.file_config.loading = false;
                  resolve(res.data)

              }).catch((err)=>{
                this.$notify({
                  title: '取消',
                  message: ''
                });
                this.pushed_pifile_configc_config.loading = false;
                reject(err.data)
              })
            }
          }).catch(()=>{
            this.$notify({
                title: ' 失败',
                message: '请求出错'
            });
            this.file_config.loading = false;
          })
        })
    },
    __delete_file_group:function(idx){
      console.log(idx)
      this.file_config.groups.splice(idx,idx+1);
    },
    __add_file_group:function(){
      this.$prompt('请输入群组名称', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          inputPattern: /[\w]+/,
          inputErrorMessage: '格式不正确'
        }).then(({ value }) => {
          this.file_config.groups.push({
            name:value,
            list:[]
          })
          this.$message({
            type: 'success',
            message: '新建群组: ' + value
          });
        }).catch(() => {
          this.$message({
            type: 'info',
            message: '取消输入'
          });       
        });
    },
    __encode:function(str) {
        // first we use encodeURIComponent to get percent-encoded UTF-8,
        // then we convert the percent encodings into raw bytes which
        // can be fed into btoa.
        return btoa(encodeURIComponent(str).replace(/%([0-9A-F]{2})/g,
            function toSolidBytes(match, p1) {
                return String.fromCharCode('0x' + p1);
            }));
    },
    __decode:function(str) {
        // Going backwards: from bytestream, to percent-encoding, to original string.
        return decodeURIComponent(atob(str).split('').map(function (c) {
            return '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2);
        }).join(''));
    },
    colorHex:function(color){
      var reg = /^(rgb|RGB)/;
      if (reg.test(color)) {
        var strHex = "#";
        // 把RGB的3个数值变成数组
        var colorArr = color.replace(/(?:\(|\)|rgb|RGB)*/g, "").split(",");
        // 转成16进制
        for (var i = 0; i < colorArr.length; i++) {
          var hex = Number(colorArr[i]).toString(16);
          if (hex === "0") {
            hex += hex;
          }
          strHex += hex;
        }
        return strHex;
      } else {
        return String(color);
      }
    },
    colorRgb:function (color) {
      // 16进制颜色值的正则
      var reg = /^#([0-9a-fA-f]{3}|[0-9a-fA-f]{6})$/;
      // 把颜色值变成小写
      color = color.toLowerCase();
      if (reg.test(color)) {
        // 如果只有三位的值，需变成六位，如：#fff => #ffffff
        if (color.length === 4) {
          var colorNew = "#";
          for (let i = 1; i < 4; i += 1) {
            colorNew += color.slice(i, i + 1).concat(color.slice(i, i + 1));
          }
          color = colorNew;
        }
        // 处理六位的颜色值，转为RGB
        var colorChange = [];
        for (let i = 1; i < 7; i += 2) {
          colorChange.push(parseInt("0x" + color.slice(i, i + 2)));
        }
        return "RGB(" + colorChange.join(",") + ")";
      } else {
        return color;
      }
    },
    getElementsByAttr: function(tagname, attrname, value) {
      let eles = document.getElementsByTagName(tagname)
      for(let i = 0; i <= eles.length; i ++){
        if(eles[i].getAttribute(attrname)==value){
          return eles[i]
        }
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss" rel="stylesheet/scss" scoped>
#app_{
  background:#EEE;
  position: absolute;
  height:100vh;
  width:100vw;
  overflow: hidden;
  #main{
    position: absolute;
    height: 100%;
    background:white;
    width: 100%;
    transition:ease all .5s;
  }
  #main.position_mode{
    transform: scale(0.85) translate(-7%, -7%);
    border-radius: 10px;
    box-shadow: 0px 0px 10px rgba(0,0,0,.1);
  }
  #center{
    height:90vh;
    width:90vw;
    position: absolute;
    top:0px;
    left:0px;
    background:transparent;
    z-index: 2;
    display: none;
  }
  #center.position_mode{
    display: block;
  }
  #right{
        height: 100vh;
      width: 10vw;
      background: transparent;
      position: absolute;
      right: -10vw;
      top: 0px;
      z-index: 2;
      transition:ease all .5s;
      .el-tag{
        font-size: 1.2rem;
        text-align: right;
        width: calc(100% - 10px) ;
        margin:5px 0px;
        ::v-deep .el-tag__close{
          
        }
      }
  }
  #right.position_mode{
    right:0px;
  }
  #bottom{
      height: 10vh;
      width: 90vw;
      background: transparent;
      position: absolute;
      left: 0px;
      bottom: -10vh;
      z-index: 2;
      transition:ease all .5s;
      padding: 0px 20px;
  }
  #bottom.position_mode{
    bottom:0px;
  }
}
.close-btn:hover{
  background-color: #f56c6c;
}
.main{
  position:fixed;
  top:0px;
  left: 0px;
  height:100%;
  width:100%;
}
.header_item{
  height: 60px;
  width: 60px;
  font-weight: bold;
  float:left;
  .header_btn{
    height: 30px;
    width: calc(100% - 10px);
    text-align: center;
    border-radius: 5px;
    border: 1px solid #ddd;
    line-height: 26px;
    box-shadow: 0px 2px 10px -2px rgba(0,0,0,.2);
    transition: ease .5s;
    margin: 5px;
    padding: 0px 5px;
    box-sizing: border-box;
    white-space: nowrap;overflow: hidden;text-overflow: ellipsis;
  }
  .header_name{
    height:15px;
    width: 100%;
    text-align: center;
    line-height: 15px;
    font-size: .5em;
  }
}
.header_item .header_btn:hover{
  box-shadow: 0px 2px 10px -4px rgba(0,0,0,.4);
}
.header_item.enable{
  cursor: pointer;
}
.header_item.disable{
  cursor:not-allowed;
  color:#aaa;
}
.big_board{
  width: 100vw;
  height: calc(100% - 60px);
  top: 60px;
  position: absolute;
  text-align: center;
  .behind_{
    width: 100%;
    position: absolute;
    height: 100%;
    z-index: 0;
    backdrop-filter: saturate(180%) blur(5px)
  }
}
.config_board{
    height: calc(100% - 70px);
    width: 30%;
    min-width: 200px;
    position: fixed;
    right: 5px;
    top: 65px;
    border-radius: 10px;
    background: rgba(255,255,255,.75);
    box-shadow: 0px 0px 10px rgba(0,0,0,.2);
    -webkit-backdrop-filter: saturate(180%) blur(20px);
    backdrop-filter: saturate(180%) blur(20px);
    padding: 10px;
    box-sizing: border-box;
    overflow: auto;
}
.board_title{
    font-size: 1.2em;
    font-weight: bold;
    padding: 10px;
    border-bottom: 1px solid #ddd;
    text-align: left;
    margin-bottom: 10px;
}
.btn{
    background: #f44336;
    margin: 10px 0px;
    color: white;
    border-radius: 5px;
    padding: 10px;
    font-weight: bold;
    box-shadow: 0 2px 12px 0 rgba(0,0,0,.1);
    transition:ease .5s;
    cursor: pointer;
    display: flex;
    justify-content: space-evenly;
    height:20px;
    line-height: 20px;
}
.btn:hover{
  box-shadow: 0 5px 12px -2px rgba(0,0,0,.3);
}
.btn.green{
    background: #009688;
}
.btn.green:hover{
  box-shadow: 0 5px 20px -10px #009688;
}
.btn.blue{
    background: #0d47a1;
}
.btn.blue:hover{
  box-shadow: 0 5px 20px -10px #0d47a1;
}
.file_config_board{
    left: 5px;
    width: calc(100% - 10px);
    background: rgba(255,255,255,.75);
    box-shadow: 0px 0px 10px rgba(0,0,0,.2);
    -webkit-backdrop-filter: saturate(180%) blur(20px);
    backdrop-filter: saturate(180%) blur(20px);
    padding: 10px;
    box-sizing: border-box;
    .btn{
      width: 50px;
      margin: 5px;
    }
    .list-group{
      min-height: 20px;
    }
    .file_group_details{
          position: absolute;
    height: 50%;
    width: calc(100% - 20px);
    min-width: 400px;
    background: white;
    z-index: 2;
    top: calc(20%);
    left: 10px;
    margin: auto;
    padding: 10px;
    box-sizing: border-box;
    border-radius: 10px;
    box-shadow: 0px 5px 20px -3px rgb(0 0 0 / 20%);
    }
    .group_item,
    .file_item{
      width: calc(25% - 15px);
        float:left;
          height: 50px;
          line-height: 50px;
          margin:5px;
          background: white;
          border-radius: 5px;
          border: 1px solid #ddd;
          text-align: left;
          text-indent: 10px;
          transition: ease .5s;
          cursor: pointer;
          position: relative;
          overflow: hidden;
          font-weight: bold;
          i{
            position: absolute;
            font-size: 3rem;
            line-height: 50px;
            color: rgba(0, 0, 0, 0.1);
            z-index: 0;
            right: 0;
        }
    }
    .file_item:hover{
      box-shadow: 0px 2px 20px -5px rgba(0,0,0,.2);
      border-color: #EEE;
    }
    .group_item{
      width: calc(100% - 20px);
      background: white;
      height: auto;
      padding: 5px;
      box-shadow: 0px 2px 20px -5px rgba(0,0,0,.2);
    }

}
.first_load_cover{
      position: fixed;
    height: 100%;
    width: 100%;
    top: 0px;
    left: 0px;
    background: transparent;
    -webkit-backdrop-filter: saturate(180%) blur(20px);
    backdrop-filter: saturate(180%) blur(20px);
    .first_load_board{
      padding: 20px 10px;
    position: fixed;
    width: 200px;
    top: 30%;
    left: calc(50% - 100px);
    z-index: 100;
    background: white;
    border-radius: 5px;
    box-shadow: 0px 3px 10px rgba(0,0,0,.1);
}
}
.color_selector_board{
  .color_selector{
        line-height: 40px;
    margin: 5px;
  }
}
.edit_board{
    width:400px;
    margin: 20px auto;
    position: relative;
}


.fullscreen_image{
  position: fixed;
  z-index: 10;
  top:0px;
  left:0px;
  height: 100vh;
  width: 100vw;
  background: rgba(0,0,0,.5);
  -webkit-backdrop-filter: saturate(180%) blur(20px);
    backdrop-filter: saturate(180%) blur(20px);
      background-position: center;
    background-repeat: no-repeat;
    background-size: contain;
    cursor: zoom-out;
}
.md_node_note{
    position: absolute;
    max-height: calc(80% - 60px);
    max-width: 40%;
    top: calc(10% + 60px);
    left: 10px;
    box-shadow: 10px 10px 20px -5px rgb(0 0 0 / 10%);
    background: white;
    border-radius: 10px;
    text-align: justify;
    padding: 5px 10px;
    box-sizing: border-box;
    word-wrap: break-word;
    word-break: normal;
    overflow: auto;
    border: 1px solid #eee;
    padding-bottom: 20px;
    font-family: 'KaTeX_Main';
}

.md_node_note.markdown_mode{
      width: calc(35% - 20px);
    max-width: calc(35% - 20px);
    top: 70px;
    height: calc(100% - 80px);
    max-height: calc(100% - 80px);
    overflow: scroll;
}
.preview_pic_note{
    position: absolute;
    max-height: calc(80% - 60px);
    max-width: 30%;
    top: calc(10% + 60px);
    left: 10px;
    box-shadow: 10px 10px 20px -5px  rgba(0,0,0,.1);
    background: white;
    border-radius: 10px;
    text-align: justify;
    padding: 0px;
    box-sizing: border-box;
    // text-indent: 20px;
    word-wrap: break-word;
    word-break: normal;
}
.el-textarea.markdown_mode{
      position: fixed;
    top: 50px;
    right: 0px;
    width: 100%;
    height: calc(100% - 50px);
    z-index:2;
    ::v-deep .el-textarea__inner{
      height: 100% !important;
      overflow: scroll;
    }
}
#header{
  position: absolute;
  z-index: 0;
  background: white;
  box-shadow: 0px 2px 10px -2px rgb(0 0 0 / 10%);
  top: 0px;
  left: 0px;
  width: 100vw;
  transition: ease .3s;
}
#md_container{
  width: 1050px !important;
  height: calc(100vh - 80px) !important;
  margin: auto;
  margin-top:20px;
  box-shadow: 0px 0px 10px -2px rgb(0,0,0,.1);
  border-radius: 0px;
  border-top-left-radius: 10px;
  border-top-right-radius: 10px;
}
@media screen and (max-width: 1050px) { /*当屏幕尺寸小于600px时，应用下面的CSS样式*/
  #md_container{
    width: 100% !important;
    height: calc(100vh - 60px) !important;
    margin: auto;
    margin-top:0px;
    border-top-left-radius: 0px;
    border-top-right-radius: 0px;
  }
}
.dark_mode{
  background:#333 !important;
  #main{
    background:#222 !important;
  }
  #header{
    background:#333 !important;
  }
  .header_btn{
    background: #444;
    color:white;
    border: 1px solid #666;
  }
  .header_name{
    color: white;
  }
  .header_item.disable{
    .header_btn{
      border: 1px solid #444;
      color:#666;
    }
    .header_name{
      color: #666;
    }
  }
  // #antv_container{
  //   background: #222 !important;
  // }
  .config_board{
    background:rgba(0,0,0,.75);
    color:white;
    .el-input{
      ::v-deep input{
        background: #666;
        border: 1px solid #444;
        color: white;
      }
    }
    .el-textarea{
      ::v-deep textarea{
        background: #666;
        border: 1px solid #444;
        color: white;
      }
    }
    .board_title{
      color:white;
      border-bottom: 1px solid #444;
    }
    .el-card{
      border:1px solid #444;
      ::v-deep .el-card__header{
        background: #333;
        border-bottom: 1px solid #444;
        color:white;
      }
      ::v-deep .el-card__body{
        background: #333;
        color:white;
      }
    }
  }
  .file_config_board .file_item{
    background: #333;
    border:1px solid #444;
  }
  .md_node_note{
        background: #333;
    color: white;
    border-color: #333;
  }
  svg g path{
    stroke:white !important;
  }
}
</style>
<style lang="scss" rel="stylesheet/scss">
// #md_container.full_screen{
//     // position: fixed !important;
//     // top: 0px !important;
//     // left: 0px !important;
//     // height: 100vh !important;
//     // width: 100vw !important;
//     // margin: 0px;
//     // border-radius: 0px;
//     // border: none;
//     // .vditor-outline{
//     //   display: none !important;
//     // }
//     // pre.vditor-reset{
//     //   padding: 20px calc(50vw - 400px) !important;
//     // }
// }
// .full_screen_button{
//   position: fixed;
//     z-index: 10;
//     opacity: 0;
// }
// .full_screen_button:hover{
//   opacity: 1;
// }
.vditor-ir__preview{
  font-size:0.8rem !important;
}
.vditor--dark.full_screen{
    background: #24292e;
}
.vditor{
  .vditor-toolbar{
    display: none;
  }
  .vditor-reset{
    padding-bottom: 40%  !important;
  }
  .vditor-toc:before{
    content: "Contents";
    font-size: 1.3rem;
    color: black;
    font-weight: bold;
    text-align: center;
    width: 100%;
    position: absolute;
    top: 5px;
    top: 5px;
    left: 0px;
    margin: 0px;
  }
  .vditor-toc{
    color: black;
    border: 1px solid;
    padding: 10px;
    padding-top:40px;
    position: relative;
    > ul{counter-reset: toc_h1;> li:before{counter-increment:toc_h1;content:counter(toc_h1) ". ";padding: 0px;font-size: 1rem;float: none;}> li{ font-weight: bold;> ul{counter-reset: toc_h2;> li:before{counter-increment:toc_h2;content:counter(toc_h1) "."counter(toc_h2) ". ";padding: 0px;font-size: 0.8rem;float: none;}> li{font-weight: normal;font-size:0.8rem;> ul{counter-reset: toc_h3;> li:before{counter-increment:toc_h3;content:counter(toc_h1) "."counter(toc_h2) "."counter(toc_h3) ". ";padding: 0px;font-size: 0.8rem;float: none;}> li{> ul{counter-reset: toc_h4;> li:before{counter-increment:toc_h4;content:counter(toc_h1) "."counter(toc_h2) "."counter(toc_h3)"."counter(toc_h4) ". ";padding: 0px;font-size: 0.8rem;float: none;}> li{> ul{counter-reset: toc_h5; > li:before{counter-increment:toc_h5;content:counter(toc_h1) "."counter(toc_h2) "."counter(toc_h3)"."counter(toc_h4)"."counter(toc_h5) ". ";padding: 0px;font-size: 0.8rem;float: none;}> li{> ul{counter-reset: toc_h6;> li:before{counter-increment:toc_h6;content:counter(toc_h1) "."counter(toc_h2) "."counter(toc_h3)"."counter(toc_h4)"."counter(toc_h5)"."counter(toc_h6) ". ";padding: 0px;font-size: 0.8rem;float: none;}}}}}}}}} }}} 
  }
  .vditor-content{
    counter-reset: h1;
    *{font-family: serif;}
    hr{    background-color: transparent;border-top: 1px solid black;}
    h1:before, h2:before, h3:before, h4:before, h5:before, h6:before{
      color:black;
    }
    h1{counter-reset:h2;border-bottom:none;}
    h2{counter-reset:h3;border-bottom:none;}
    h3{counter-reset:h4}h4{counter-reset:h5}h5{counter-reset:h6}  h1:before{counter-increment:h1;content:counter(h1) ". ";padding: 0px;font-size: 1rem;float: none;} h2:before{counter-increment:h2;content:counter(h1) "." counter(h2) ". ";padding: 0px;font-size: 1rem;float: none;}  h3:before,h3.md-focus.md-heading:before{counter-increment:h3;content:counter(h1) "." counter(h2) "." counter(h3) ". ";padding: 0px;font-size: 1rem;float: none;}  h4:before,h4.md-focus.md-heading:before{counter-increment:h4;content:counter(h1) "." counter(h2) "." counter(h3) "." counter(h4) ". ";padding: 0px;font-size: 1rem;float: none;}  h5:before,h5.md-focus.md-heading:before{counter-increment:h5;content:counter(h1) "." counter(h2) "." counter(h3) "." counter(h4) "." counter(h5) ". ";padding: 0px;font-size: 1rem;float: none;}  h6:before,h6.md-focus.md-heading:before{counter-increment:h6;content:counter(h1) "." counter(h2) "." counter(h3) "." counter(h4) "." counter(h5) "." counter(h6) ". "} >h3.md-focus:before, >h4.md-focus:before, >h5.md-focus:before, >h6.md-focus:before,h3.md-focus:before,h4.md-focus:before,h5.md-focus:before,h6.md-focus:before{color:inherit;border:inherit;border-radius:inherit;position:inherit;left:initial;float:none;top:initial;font-size:inherit;padding-left:inherit;padding-right:inherit;vertical-align:inherit;font-weight:inherit;line-height:inherit;}
  }
}
.vditor--dark{
  .vditor-content{
    hr{border-top: 1px solid white !important;}
    h1:before, h2:before, h3:before, h4:before, h5:before, h6:before{
      color:white;
    }
    p.head_title{
      color:white;
      border-bottom: 4px double white !important;
    }

  }
  .vditor-toc{
    color: white;
    
    }
    .vditor-toc::before{
      color: white;
    }
}
@media screen and (max-width: 600px) { /*当屏幕尺寸小于600px时，应用下面的CSS样式*/
  .vditor-outline{display: none !important;}
}

</style>