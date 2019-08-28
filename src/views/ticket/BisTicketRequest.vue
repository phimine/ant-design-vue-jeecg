<template>
<div>
  <div class="ant-alert ant-alert-info table-operator" style="margin-bottom: 16px;" v-show="this.ticketId">
    当前任务环节：{{ this.currentAction }}
  </div>
  <a-card :bordered="false">

    <a-form @submit="handleSubmit" :form="form">
      <a-form-item label="申请人" :labelCol="{ span: 7 }" :wrapperCol="{ span: 15 }" v-show="this.ticketId">
        <a-input v-model="ticket.drafter" v-decorator="['drafter',{}]" :disabled="this.ticketId"/>
      </a-form-item>
      <a-form-item label="请示类型" :labelCol="{ span: 7 }" :wrapperCol="{ span: 15 }">
        <a-select v-decorator="['type',{rules: [{ required: !this.ticketId, message: '请选择请示类型!' }]}]"
                  placeholder="请选择请示类型"
                  :disabled="this.ticketId"
                  v-model="ticket.type">
          <a-select-option v-for="(item,index) in typeOptions" :key="index" :value="item.value">{{ item.text }}</a-select-option>
        </a-select>
      </a-form-item>

      <a-form-item label="标题" :labelCol="{ span: 7 }" :wrapperCol="{ span: 15 }">
        <a-input v-model="ticket.title" v-decorator="['title',{rules: [{ required: !this.ticketId, message: '请输入标题!' }]}]" :disabled="this.ticketId"/>
      </a-form-item>

      <a-form-item label="紧急程度" :labelCol="{ span: 7 }" :wrapperCol="{ span: 15 }">
        <a-select v-model="ticket.priority" v-decorator="['priority',{rules: [{ required: !this.ticketId, message: '请选择紧急程度!' }]}]" placeholder="请选择紧急程度" :disabled="this.ticketId">
          <a-select-option v-for="(item,index) in priorityOptions" :key="index" :value="item.value">{{ item.text }}</a-select-option>
        </a-select>
      </a-form-item>

      <a-form-item label="上传请示正文" :labelCol="{ span: 7 }" :wrapperCol="{ span: 15 }" v-show="!this.ticketId">
        <a-upload class="avatar-uploader"
                  :action="uploadAction"
                  :beforeUpload="beforeUpload"
                  :headers="headers"
                  @change="handleUploadChange"
                  >
          <a-button>
            <a-input type="text" hidden v-model="ticket.content" v-decorator="['content',{rules: [{ required: false }]}]" />
            <a-icon type="upload" class="ant-upload-text" /> Upload
          </a-button>
        </a-upload>
      </a-form-item>

      <a-form-item label="备注" :labelCol="{ span: 7 }" :wrapperCol="{ span: 15 }">
        <a-input v-model="ticket.remark" v-decorator="['remark',{rules: [{ required: false }]}]" :disabled="this.ticketId"/>
      </a-form-item>

      <a-form-item label="填写意见" :labelCol="{ span: 7 }" :wrapperCol="{ span: 15 }">
        <a-textarea v-model="ticket.comment" v-decorator="['comment',{rules: [{ required: false }]}]" placeholder="填写意见" :rows="4" />
      </a-form-item>

      <a-form-item label="会签部门" :labelCol="{ span: 7 }" :wrapperCol="{ span: 15 }" v-show="this.currentActionCode =='managerReview'">
        <a-select placeholder="请选择会签部门"
                  mode="multiple"
                  @change="handleChange">
          <a-select-option v-for="(item,index) in deptOptions" :key="index" :value="item.id">{{ item.departName }}</a-select-option>
        </a-select>
      </a-form-item>

      <a-form-item label="会签承办人" :labelCol="{ span: 7 }" :wrapperCol="{ span: 15 }"
                   v-show="needAssign()">
        <a-select placeholder="指定会签承办人"
                  @change="handleSignOwnerChange">
          <a-select-option v-for="(item,index) in signOwnerOptions" :key="index" :value="item.id">{{ item.realname }}</a-select-option>
        </a-select>
      </a-form-item>

      <a-form-item label="副总裁" :labelCol="{ span: 7 }" :wrapperCol="{ span: 15 }"
                   v-show="this.currentActionCode =='managerAdapt'">
        <a-select placeholder="指定副总裁审批"
                  @change="handleVPChange">
          <a-select-option v-for="(item,index) in vpOptions" :key="index" :value="item.id">{{ item.realname }}</a-select-option>
        </a-select>
      </a-form-item>

      <a-form-item :wrapperCol="{ span: 12, offset: 19 }">

        <div class="table-operator">
          <a-button type="primary" icon="save" @click="handleSave" v-show="!this.ticketId">保存</a-button>
          <a-button type="primary" icon="check" @click="handleSubmit" v-show="!this.ticketId">提交</a-button>
          <a-button type="primary" v-for="(item, index) in outcomes" :key="index" @click="handleNext(item.conditionValue)" >{{item.name}}</a-button>

        </div>
      </a-form-item>
    </a-form>
  </a-card>
</div>
</template>

<script>
  import { getAction, postAction } from '@/api/manage'
  import Vue from 'vue'
  import { ACCESS_TOKEN } from "@/store/mutation-types"
  import {  getInfo } from '@/api/login'
  import { queryall } from '@/api/api'

  export default {
    data () {
      return {
        formLayout: 'horizontal',
        form: this.$form.createForm(this),
        headers:{},
        typeOptions:[],
        priorityOptions:[],
        deptOptions:[],
        selectedDepts:[],
        signOwnerOptions:[],
        vpOptions:[],
        nextSignOwner:"",
        signManagers:"",
        startAssign: false,
        assignNeeded: true,
        vp:"",
        userId:"",
        username:"",
        ticketId:"",
        ticket:{},
        currentAction:"",
        currentActionCode:"",
        outcomes:[],
        roleList:[],
        url: {
          imgerver: window._CONFIG['domianURL']+"/sys/common/view",
          pdferver: window._CONFIG['domianURL']+"/sys/common/pdf/pdfPreviewIframe",
          saveTicket: window._CONFIG['domianURL']+"/ticket/bisTicket/draft",
          submitTicket: window._CONFIG['domianURL']+"/ticket/bisTicket/submit",
          operate: window._CONFIG['domianURL']+"/ticket/bisTicket/operate",
          currentAct: window._CONFIG['domianURL']+"/ticket/bisTicket/currentAct",
          outcomes: window._CONFIG['domianURL']+"/ticket/bisTicket/outcomes",
          getTicket: window._CONFIG['domianURL']+"/ticket/bisTicket/queryById",
          signDeptList: window._CONFIG['domianURL']+"/ticket/bisTicket/signDeptList",
          signDeptManagers: window._CONFIG['domianURL']+"/ticket/bisTicket/signDeptManagers",
          validSignTaker: window._CONFIG['domianURL']+"/ticket/bisTicket/validSignTaker",
          assignNeeded: window._CONFIG['domianURL']+"/ticket/bisTicket/assignNeeded",
          vpList: window._CONFIG['domianURL']+"/ticket/bisTicket/vpList",
          fileUpload: window._CONFIG['domianURL'] + "/sys/common/upload"
        }
      }
    },
    computed:{
      uploadAction:function () {
        return this.url.fileUpload;
      },
    },
    methods: {
      needAssign(){
        return (this.currentActionCode =='assignTask'
                  || this.currentActionCode =='firstSign'
                  || this.currentActionCode =='secondSign')
                && this.assignNeeded;
      },
      handleChange(value) {
        getAction(this.url.signDeptManagers, {
          departs: value.toString()
        }).then((res)=>{
          if (res.success){
            this.signManagers = res.result;
          }
        })
      },
      handleSignOwnerChange(value){
        this.nextSignOwner = value.toString();
      },
      handleVPChange(value){
        this.vp = value.toString();
      },
      /** ==========上传文件相关========== */
      beforeUpload: function(file){
        var fileType = file.type;
        // TODO 验证文件类型
        // TODO 验证文件大小
      },
      handleUploadChange (info) {
        if (info.file.status === 'uploading') {
          this.uploadLoading = true;
          return
        }
        if (info.file.status === 'done') {
          var response = info.file.response;
          this.uploadLoading = false;
          console.log(response);
          if(response.success){
            this.form.setFieldsValue({content:response.message});
          }else{
            this.$message.warning(response.message);
          }
        }
      },
      getAvatarView(){
        return this.url.pdferver +"/"+ this.form.getFieldValue("content");
      },

      /** ==========提交按钮========== */
      handleSubmit (e) {
        e.preventDefault()
        let params = {
          drafterId:this.userId,
          drafter:this.username,
          title:this.form.getFieldValue("title"),
          type:this.form.getFieldValue("type"),
          priority:this.form.getFieldValue("priority"),
          remark:this.form.getFieldValue("remark"),
          content:this.form.getFieldValue("content"),
          comment:this.form.getFieldValue("comment"),
        }
        this.form.validateFields((err, values) => {
          if (!err) {
            // TODO 验证表单

          }
        })
        postAction(this.url.submitTicket, params).then((res) => {
          // TODO 页面跳转
          this.$router.push({ path: "BisTicketList" })
        })
      },
      /** ==========保存按钮========== */
      handleSave (e) {
        e.preventDefault()
        let params = {
          drafterId:this.userId,
          drafter:this.username,
          title:this.form.getFieldValue("title"),
          type:this.form.getFieldValue("type"),
          priority:this.form.getFieldValue("priority"),
          remark:this.form.getFieldValue("remark"),
          content:this.form.getFieldValue("content"),
          comment:this.form.getFieldValue("comment"),
        }
        this.form.validateFields((err, values) => {
          if (!err) {
            // TODO 验证表单

          }
        })
        postAction(this.url.saveTicket, params).then((res) => {
          // TODO 页面跳转
          this.$router.push({ path: "TaskList" })
        })
      },
      /** ==========公用按钮========== */
      handleNext(value){
        let params = {
          ticketId:this.ticketId,
          takerId:this.userId,
          comment:this.form.getFieldValue("comment"),
          variable:value,
          nextTakerIds:""
        }
        if (this.currentActionCode == 'managerReview'){
          if (value == 'true'){
            if (!this.signManagers){
              this.$message.warning("请选择会签部门！");
              return;
            }
            params.nextTakerIds = this.signManagers;
          } else {
            this.startAssign = false;
          }
        } else if (this.currentActionCode =='assignTask'
                    || this.currentActionCode =='firstSign'
                    || this.currentActionCode =='secondSign'){
          if (value == 'false'){
            if (!this.nextSignOwner){
              this.$message.warning("请选择会签承办人！");
              return;
            }
            params.nextTakerIds = this.nextSignOwner;
          }
        } else if (this.currentActionCode =='managerAdapt'){
          if (value == 'true'){
            if (!this.vp){
              this.$message.warning("请选择副总裁！");
              return;
            }
            params.nextTakerIds = this.vp;
          }
        }
        postAction(this.url.operate, params).then((res) => {
          if (res.success) {
            this.$router.push({ path: "TaskList" })
          } else {
            this.$message.warning(res.result);
          }
        })
      },

      getParams(){
        this.ticketId = this.$route.query.ticketId
      }
    },
    created (){
      const token = Vue.ls.get(ACCESS_TOKEN);
      this.headers = {"X-Access-Token":token};

      // this.initialRoleList();

      this.getParams();
      /** ==========请示详情========== */
      getAction(this.url.getTicket, {
        id:this.ticketId
      }).then((res)=>{
        if(res.success){
          this.form.setFieldsValue({
            drafter:res.result.drafter,
            title:res.result.title,
            content:res.result.content,
            type:res.result.type,
            priority:res.result.priority,
            remark:res.result.remark
          });
        }
      });

      /** ==========vpList========== */
      getAction(this.url.vpList).then((res)=>{
        if(res.success){
          this.vpOptions = res.result;
        }
      });

      /** ==========会签部门列表========== */
      getAction(this.url.signDeptList).then((res)=>{
        if (res.success){
          this.deptOptions = res.result;
        }
      });

      /** ==========用户信息========== */
      getInfo().then((res)=>{
        if(res.success){
          this.userId = res.result.id;
          this.username = res.result.realname;

          /** ==========当前活动节点========== */
          getAction(this.url.currentAct, {
            ticketId:this.ticketId,
            takerId:this.userId
          }).then((res)=>{
            if(res.success){
              this.currentAction = res.result.actionName;
              this.currentActionCode = res.result.actionKey;
            }
          });
          /** ==========当前活动连线走向========== */
          getAction(this.url.outcomes, {
            ticketId:this.ticketId,
            takerId:this.userId
          }).then((res)=>{
            if(res.success){
              // this.outcomes = Object.assign({}, res.result)
              this.outcomes = res.result;
            }
          });
          /** ==========会签承办人列表========== */
          getAction(this.url.validSignTaker, {
            ticketId:this.ticketId,
            takerId:this.userId
          }).then((res)=>{
            if(res.success){
              this.signOwnerOptions = res.result;
            }
          });
          /** ==========根据会签状态决定是否需要指定下一承办人========== */
          getAction(this.url.assignNeeded, {
            ticketId:this.ticketId,
            takerId:this.userId
          }).then((res)=>{
            if(res.success){
              this.assignNeeded = res.result;
            }
          });
        }
      });

      /** ==========请示类型========== */
      // TODO 动态获取
      this.typeOptions = [{
        text:"请示类型1",
        value:"0"
      },{
        text:"请示类型2",
        value:"1"
      },{
        text:"请示类型3",
        value:"2"
      }];

      this.priorityOptions = [{
        text:"正常",
        value:"2"
      },{
        text:"紧急",
        value:"1"
      },{
        text:"非常紧急",
        value:"0"
      }];
    },
    watch: {
      '$route': 'getParams'
    }
  }
</script>