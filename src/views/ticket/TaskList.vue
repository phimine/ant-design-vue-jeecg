<template>
  <a-card :bordered="false">
    <div>
      <a-table bordered :dataSource="dataSource" :columns="columns">
        <span slot="type" slot-scope="text">
        {{ text | typeFilter }}
        </span>
        <span slot="priority" slot-scope="text">
        {{ text | priorityFilter }}
        </span>
        <span slot="status" slot-scope="text">
        {{ text | statusFilter }}
        </span>

        <template slot="action" slot-scope="text, record">
          <a @click="handleView(record)">查看</a>
        </template>
      </a-table>

      <a-modal
        title="查看"
        :width="800"
        v-model="visible"
        @ok="handleOk"
      >
        <a-form :autoFormCreate="(form)=>{this.form = form}">

          <a-form-item
            :labelCol="labelCol"
            :wrapperCol="wrapperCol"
            label="标题"
            hasFeedback
          >
            <a-input v-model="modalData.title" id="no" disabled="disabled" />
          </a-form-item>

          <a-form-item
            :labelCol="labelCol"
            :wrapperCol="wrapperCol"
            label="申请人"
            hasFeedback
          >
            <a-input v-model="modalData.drafter" id="permission_name" disabled="disabled" />
          </a-form-item>

          <a-form-item
            :labelCol="labelCol"
            :wrapperCol="wrapperCol"
            label="类型"
            hasFeedback
          >
            <a-select v-model="modalData.type" disabled="disabled" >
              <a-select-option value="0">请假</a-select-option>
              <a-select-option value="1">报销</a-select-option>
              <a-select-option value="2">出差</a-select-option>
            </a-select>
          </a-form-item>

          <a-form-item
            :labelCol="labelCol"
            :wrapperCol="wrapperCol"
            label="紧急程度"
            hasFeedback
          >
            <a-select v-model="modalData.priority" disabled="disabled" >
              <a-select-option value="0">非常紧急</a-select-option>
              <a-select-option value="1">紧急</a-select-option>
              <a-select-option value="2">正常</a-select-option>
            </a-select>
          </a-form-item>

          <a-form-item
            :labelCol="labelCol"
            :wrapperCol="wrapperCol"
            label="内容"
            hasFeedback
          >
            <a-input v-model="modalData.content" id="content" disabled="disabled" />
          </a-form-item>

        </a-form>
      </a-modal>
    </div>
  </a-card>
</template>

<script>
  import STable from '@/components/table/'
  import { getAction } from '@/api/manage'
  import {  getInfo } from '@/api/login'

  export default {
    name: "TaskList",
    data () {
      return {
        visible: false,
        labelCol: {
          xs: { span: 24 },
          sm: { span: 5 },
        },
        wrapperCol: {
          xs: { span: 24 },
          sm: { span: 16 },
        },
        form: null,
        mdl: {},
        dataSource: [],
        userId:"",
        username:"",

        // 高级搜索 展开/关闭
        advanced: false,


        modalData: {},
        // 表头
        columns: [
          {
            title: '标题',
            dataIndex: 'title'
          },
          {
            title: '申请人',
            dataIndex: 'drafter',
          },
          {
            title: '类型',
            dataIndex: 'type',
            scopedSlots: { customRender: 'type' },
          },
          {
            title: '紧急程度',
            dataIndex: 'priority',
            scopedSlots: { customRender: 'priority' },
          },
          {
            title: '状态',
            dataIndex: 'status',
            scopedSlots: { customRender: 'status' },
          },
          {
            title: '操作',
            width: '150px',
            dataIndex: 'action',
            scopedSlots: { customRender: 'action' },
          }
        ],
        url: {
          taskList: window._CONFIG['domianURL']+"/ticket/bisTicket/taskList",
          fileUpload: window._CONFIG['domianURL'] + "/sys/common/upload"
        },
      }
    },
    filters: {
      statusFilter(status) {
        const statusMap = {
          "0": "已起草",
          "1": "已提交",
          "2":"提交审核中",
          "21":"审核退回",
          "22":"审核通过",
          "3":"会签中",
          "4":"会签完成",
          "5":"等待调整",
          "52":"已调整",
          "6":"批阅中",
          "61":"退回调整",
          "7":"已批准",
          "8":"已拒绝"
        }
        return statusMap[status]
      },
      typeFilter(type) {
        const typeMap = {
          "0": "请假",
          "1": "报销",
          "2":"出差"
        }
        return typeMap[type]
      },
      priorityFilter(priority) {
        const priorityMap = {
          "0": "非常紧急",
          "1": "紧急",
          "2":"正常"
        }
        return priorityMap[priority]
      }


    },
    created () {
      getInfo().then((res)=>{
        if(res.success){
          this.userId = res.result.id;
          this.username = res.result.realname;
          console.log("获取用户信息成功");
          console.log("userId================" +this.userId);
          var params = {userId: this.userId};
          console.log("params================" +params);
          getAction(this.url.taskList, params).then(res => {
            console.log("result===================" + res);
            this.dataSource = res.result;
          })
        }
      });
    },
    methods: {
      handleView(record){
        // this.modalData = Object.assign({}, record)
        // this.visible = true

        console.log(record)
        this.$router.push({
          path: "BisTicketRequest",
          query: {
            ticketId: record.id
          }
        },)
      },
      handleOk () {
        this.visible = false
      },
      toggleAdvanced () {
        this.advanced = !this.advanced
      },
    },
    watch: {
      /*
      'selectedRows': function (selectedRows) {
        this.needTotalList = this.needTotalList.map(item => {
          return {
            ...item,
            total: selectedRows.reduce( (sum, val) => {
              return sum + val[item.dataIndex]
            }, 0)
          }
        })
      }
      */
    }
  }
</script>