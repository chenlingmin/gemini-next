<template>
      <PageHeader :title="$t('common.profile.title')" :subTitle="$t('common.profile.subtitle')"></PageHeader>
      <a-card>
            <a-row type="flex" justify="center" align="middle">
                  <a-col :span="10">
                        <a-form ref="formRef" :model="formItem" layout="vertical" :rules="rules">
                              <a-form-item :label="$t('user.form.user')">
                                    <a-input v-model:value="store.state.user.account.user" disabled></a-input>
                              </a-form-item>
                              <a-form-item :label="$t('user.form.dept')">
                                    <a-input v-model:value="formItem.department"></a-input>
                              </a-form-item>
                              <a-form-item :label="$t('user.form.real_name')">
                                    <a-input v-model:value="formItem.real_name"></a-input>
                              </a-form-item>
                              <a-form-item :label="$t('user.form.email')">
                                    <a-input v-model:value="formItem.email"></a-input>
                              </a-form-item>
                              <a-form-item :label="$t('common.theme')">
                                    <a-select @change="changeTheme" v-model:value="formItem.theme">
                                          <a-select-option key="dark" value="dark">暗黑主题</a-select-option>
                                          <a-select-option key="light" value="light">普通主题</a-select-option>
                                    </a-select>
                              </a-form-item>
                              <a-form-item :label="$t('user.password.new')" name="password">
                                    <a-input-password v-model:value="formItem.password"></a-input-password>
                              </a-form-item>
                              <a-form-item :label="$t('user.password.confirm')" name="confirm_password">
                                    <a-input-password v-model:value="formItem.confirm_password"></a-input-password>
                              </a-form-item>
                              <a-form-item>
                                    <a-button @click="() => request.Edit(formItem, false)" block type="primary">{{
                                                $t('common.save')
                                    }}</a-button>
                              </a-form-item>
                        </a-form>
                  </a-col>
            </a-row>
      </a-card>

</template>

<script lang="ts" setup>
import { useStore } from '@/store';
import { onMounted, ref } from 'vue';
import CommonMixins from "@/mixins/common"
import PageHeader from '@/components/pageHeader/pageHeader.vue';
import { RuleObject } from 'ant-design-vue/lib/form/interface';
import { RegisterForm, Request } from '@/apis/user';
import { AxiosResponse } from 'axios';
import { Res } from '@/config/request';

const store = useStore()

const formItem = ref<RegisterForm>({
      password: "",
      confirm_password: "",
      real_name: "",
      email: "",
      department: "",
      theme: "dark"
})

const changeTheme = (e) => {
      localStorage.setItem("theme", e)
      location.reload();
}


const validPassword = async (rule: RuleObject, value: string) => {
      if (value !== formItem.value.password && value !== "") {
            return Promise.reject('输入的密码不一致')
      } else {
            return Promise.resolve();
      }
};

const request = new Request()

const { regExpPassword } = CommonMixins()

const rules = {
      password: [{ validator: regExpPassword, trigger: 'blur', required: true }],
      confirm_password: [{ trigger: 'blur', message: "请确认密码", required: true }, { required: true, validator: validPassword, trigger: 'blur' }],

}


onMounted(() => {
      request.UserInfo().then((res: AxiosResponse<Res<RegisterForm>>) => {
            formItem.value = res.data.payload
            localStorage.getItem("theme") === null ? formItem.value.theme = "dark" : formItem.value.theme = localStorage.getItem("theme") as string
      })
})

</script>