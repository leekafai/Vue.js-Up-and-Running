#### 第六章
----
### 使用 Vuex 进行状态管理

在阅读本章节前，所有的数据应已存储在我们的组件中。当我们请求一个接口，其返回的数据将存储在 data 对象中。当我们将表单与一个值绑定后，我们也会将该值存储在 data 对象中。
组件间的通讯，可以通过 events（子组件到父组件）和 props（父组件到子组件）来实现。这些方式尚能应付简单的情况，但在更为复杂的应用中则力不能支。

举一个社交应用中的最具代表性的案例：信息功能。你希望顶部导航栏上通过一个图标显示你的通知数量，然后你希望底部能够弹出一个提示消息，消息中能够告诉你，你的通知数量是多少。这两个组件在页面中可以说毫无关系，如果此时使用 events 和 props 将其关联，会令开发徒增痛苦：这些与通知本身完全无关的组件，将要对通知事件进行监听。另一种方法是， 将 API 的请求分离到各个组件上，而不是将这些组件关联到共享的数据上。但这么做只会更糟！组件的更新可能不同步，意味着它们有可能会显示不同的东西，同时也会产生冗余的网络请求。

vuex 是一个帮助开发者管理 Vue 应用状态的库。它采用的集中式存储，令你可在应用中存储和使用全局状态，并能对即将存储的数据进行验证，确保你能取出预料的正确数据。

### 安装方式

你可以通过 CDN 来使用 vuex。只需添加如下内容：

```html
<script src="https://unpkg.com/vuex"></script>
```

另外，若你正在使用 npm，你可以执行`npm install --save vuex`指令来使用 vuex。
如果你使用了打包工具，例如 webpack，如同使用 vue-router 一样，你需要执行 `Vue.use()`：

```javascript
import Vue from 'vue';
import Vuex from 'vuex';
Vue.use(Vuex);
```

接下来，你需要设置你的 store。将下面的代码保存到 **store/index.js** 文件中：

```javascript
import Vuex from 'vuex';
export default new Vuex.Store({
	state: {}
});
```
目前为止，它依然是一个空的 store：我们将在本章中为其添加数据。

然后，在入口文件中引用上面创建的文件，并作为属性将其添加到 Vue 实例中。

```javascript
import Vue from 'vue';
import store from './store';
new Vue({
	el: '#app',
	store,
	components: {
		App
	}
});
```

现在，你成功在应用中添加了 store，可以通过`this.$store`去使用它。我们来看一下关于 vuex 的概念，从而学习到`this.$store`的作用。

### 概念

正如本章介绍所言，当复杂的应用中有多个组件需要共享状态时，可以使用 vuex。

这里是一个没有使用 vuex 的简单组件，在页面中用来显示用户信息：

```javascript
const NotificationCount = {
	template: `<p>Messages: {{ messageCount }}</p>`,
	data: () => ({
		messageCount: 'loading'
		}),
	mounted() {
		const ws = new WebSocket('/api/messages');
		ws.addEventListener('message', (e) => {
			const data = JSON.parse(e.data);
			this.messageCount = data.messages.length;
		});
	}
};
```

一个很简单的例子。这个组件会启动一个 websocket，地址为 **/api/messages**，每当服务器推送消息到客户端中——在这个例子中，当 socket 已连接（初始化通知数量）并当数量有更新时（有新消息），通过 socket 传输的消息在统计数量后将显示在页面中。

![Alt text](./images-leekafai/bird.jpg)
实际情况中，实现代码会更加复杂：在这个例子中，websocket缺少验证，并假设了 websocket的响应数据是正确的JSON结构，messages属性的类型是数组，而实际中并非绝对。但在这个例子里，这段简单的代码显然实现了目的。

恰逢我们想在同一个页面中，使用多于一个的通知计数组件时，问题出现了。当每一个组件都打开一个 websocket 连接时，会多出许多不必要的连接。再者，由于网络的延迟，组件的更新可能会不同步。为了解决这个问题，我们可以将 websocket 的服务逻辑放到 vuex 中。

跟着例子，马上开工。我们的组件设计将会是这样的：
```javascript
const NotificationCount = {
	template: `<p>Messages: {{ messageCount }}</p>`,
	computed: {
		messageCount() {
			return this.$store.state.messages.length;
		}
	}
	mounted() {
		this.$store.dispatch('getMessages');
	}
};
```

接下来，设置 vuex 的 store：
```javascript
let ws;
export default new Vuex.Store({
	state: {
		messages: [],
	},
	mutations: {
		setMessages(state, messages) {
			state.messages = messages;
		}
	},
	actions: {
		getMessages({ commit }) {
			if (ws) {
				return;
			}
			ws = new WebSocket('/api/messages');
			ws.addEventListener('message', (e) => {
				const data = JSON.parse(e.data);
				commit('setMessages', data.messages);
			});
		}
	}
});
```
现在，每一个通知计数组件都挂载在触发器 getMessages 上，触发器会检查 websocket 是否已经连接，如果没有连接，则会打开唯一的一个 websocket 连接，并持续监听 socket，提交状态的变更，然后令通知计数组件随 store 的更新而更新——与 Vue 的多数设定一致。当 socket 接收到数据，全局的 store 会更新，继而同时更新页面中的每一个组件。

在本章的剩余内容中，我将逐一介绍你在上述例子中遇到的概念——state，mutations 与 actions，并阐述一种能使我们妥善使用 vuex 模块的方式，从而避免在大型应用中出现杂乱、庞大的文件。

State 与 State Helpers

先来谈谈 state。**state** 是数据在 vuex store 中的存储体现。其如同一个遵循单一来源（single source of truth，又称 SSOT）、可以在应用中无阻获取的对象。

举一个存储单个数字的简单例子：

```javascript
import Vuex from 'vuex';
export default new Vuex.Store({
	state: {
		messageCount: 10
	}
});
```