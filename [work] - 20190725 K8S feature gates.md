# k8s feature gates



## Do What

### Remove Fields from object

![image-20190725184937492](/Users/alauda/Library/Application Support/typora-user-images/image-20190725184937492.png)

## Howto

### Add default features

![image-20190725192141862](/Users/alauda/Library/Application Support/typora-user-images/image-20190725192141862.png)

![image-20190725192104463](/Users/alauda/Library/Application Support/typora-user-images/image-20190725192104463.png)

![image-20190725192005435](/Users/alauda/Library/Application Support/typora-user-images/image-20190725192005435.png)

### Add flagset

![image-20190725191444140](/Users/alauda/Library/Application Support/typora-user-images/image-20190725191444140.png)

### DefaultMutableFeatureGate

![image-20190725191619566](/Users/alauda/Library/Application Support/typora-user-images/image-20190725191619566.png)

![image-20190725191647475](/Users/alauda/Library/Application Support/typora-user-images/image-20190725191647475.png)

### Add from features

![image-20190725192548822](/Users/alauda/Library/Application Support/typora-user-images/image-20190725192548822.png)

### Check if feature enabled

![image-20190725192717911](/Users/alauda/Library/Application Support/typora-user-images/image-20190725192717911.png)

- 启动时通过—feature-gates传入开启、关闭的feature
- featureGate 对象有连个属性known，enabled
- 判定某个feature是否开启的时候，首先尝试从enabled判断，如果没有则从known获取默认值

### 疑问

- enabled是如何从 命令行参数解析， 然后设置到 enabled 属性中的？