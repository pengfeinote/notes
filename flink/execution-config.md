## flink config

### Execution config

#### setBufferTimeout

flink不会将元素一个一个通过网络读进来，而是会将元素放在buffer中，然后通过网络发送，buffer一般在满了的时候才会发送，如果设置了setBufferTimeout(100)，则100ms发送一次（不管buffer是否满），如果setBufferTimeout(-1)，则只有buffer满了才会发送，默认值是10
