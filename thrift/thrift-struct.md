required，optional与默认：

required: 必须填写，不能为null

optional：为null时，对象序列化时不会序列化该字段（isset为false时，不会序列化）

默认：不必须填写，即使为null，也会序列化
