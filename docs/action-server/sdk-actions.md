# 动作

`Action` 类是任何自定义动作的基类。要定义自定义动作，请创建 `Action` 类的子类并覆盖两个必须的方法：`name` 和 `run`。动作服务器在收到执行动作的请求时，会根据其 `name` 方法的返回值调用动作。

一个自定义动作的骨架如下所示：

```python
class MyCustomAction(Action):

    def name(self) -> Text:

        return "action_name"

    async def run(
        self, dispatcher, tracker: Tracker, domain: Dict[Text, Any],
    ) -> List[Dict[Text, Any]]:

        return []
```

## 方法 {#methods}

### Action.name {#actionname}

定义动作的名称。此方法返回的名称是机器人领域中使用的名称。

返回：

动作名称

返回类型：

`str`

### Action.run {#actionrun}

```python
async Action.run(dispatcher, tracker, domain)
```

`run` 方法执行动作的副作用。

参数：

- `dispatcher`：用于将消息发送回用户的调度程序。使用 `dispatcher.utter_message()` 或任何其他 `rasa_sdk.executor.CollectingDispatcher` 方法。请参阅[调度程序文档](/action-server/sdk-dispatcher/)。
- `tracker`：当前用户的状态追踪器。可以使用 `tracker.get_slot(slot_name)` 访问槽值，最新的用户消息是 `tracker.latest_message.text` 以及任何其他 `rasa_sdk.Tracker` 属性。请参阅[追踪器文档](/action-server/sdk-tracker/)。
- `domain`：对话机器人的领域。

返回：

`rasa_sdk.events.Event` 实例的列表。请参阅[事件文档](/action-server/sdk-events/)。

返回类型：

`List[Dict[str, Any]]`

## 示例 {#example}

在餐厅对话机器人中，如果用户说“show me a Mexican restaurant”，对话机器人可以执行 `ActionCheckRestaurants` 动作，可能如下所示：

```python
from typing import Text, Dict, Any, List
from rasa_sdk import Action
from rasa_sdk.events import SlotSet

class ActionCheckRestaurants(Action):
   def name(self) -> Text:
      return "action_check_restaurants"

   def run(self,
           dispatcher: CollectingDispatcher,
           tracker: Tracker,
           domain: Dict[Text, Any]) -> List[Dict[Text, Any]]:

      cuisine = tracker.get_slot('cuisine')
      q = "select * from restaurants where cuisine='{0}' limit 1".format(cuisine)
      result = db.query(q)

      return [SlotSet("matches", result if result is not None else [])]
```

此操作查询数据库来查找与请求的菜系匹配的餐厅，并使用找到的餐厅列表来设置 `matches` 槽值。
