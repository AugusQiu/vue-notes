v-model就是v-bind:value和v-on:input的语法糖，我们在自定义的组件上使用v-model
````js
<custom-input
  v-model="searchText"
>
</custom-input>

// 等价于
<custom-input
  v-bind:value="searchText"
  v-on:input="searchText = $event"
>
</custom-input>
````