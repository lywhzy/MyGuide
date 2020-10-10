# kafka

![binaryTree](../image/1122015-20170525120743700-472655419.png)

- 同一topic的一条消息只能由同一Consumer组的Consumer消费
- 如果想使用单播，就将所有Consumer分在一个Consumer组中
- 如果想使用广播，就将每一个Consumer分在不同的Consumer组中