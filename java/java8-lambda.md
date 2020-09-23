## 本文记载java8中lambda表达式的用法及注意事项

### list -> map(key, list<value>)

list.stream().collect(Collectors.groupingby(Object::getName()));

### list -> map(key, set<value>)

list.stream().collect(Collectors.groupingby(Object::getName(). Collectors.toSet()));
