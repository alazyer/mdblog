# k8s feature gates



## Do What

### APIServer Example: Remove Fields from object

![](./assets/img/apiserver_main_func.png)![](./assets/img/newapiservercommand_func.png)

![](./assets/img/run_func.png)

![](./assets/img/createserverchain_func.png)

![](./assets/img/createkubeapiserver_func.png)



![](./assets/img/completedconfig_new_method.png)

![](./assets/img/master_installapis_method.png)

![](./assets/img/genericapiserver_installapigroups_method.png)

![](./assets/img/genericapiserver_installapiresources_method.png)

![](./assets/img/apigroupversion_installrest_method.png)

![](./assets/img/apiinstaller_install_method.png)

APIInstaller的registerResourceHandlers方法中会盗用restfulCreateResource方法

![](./assets/img/restfulcreateresource_func.png)

![](./assets/img/createresource_func.png)

![](./assets/img/namedcreateradapter_create_method.png)

![](./assets/img/store_create_method.png)

![](./assets/img/before_create_func.png)

![image-20190726140046010](./assets/img/resource_quota_strategy_prepare_for_xxx_method.png)

![resourcequota_dropdisabledfields](./assets/img/resourcequota_dropdisabledfields.png)

### Controller Example 1: Dynamic Load Plugins

![](./assets/img/newcontrollerinitializers_func.png)

#### CSIInlineVolume feature

![](./assets/img/startpersistentvolumebindercontroller_func.png)

![](./assets/img/probecontrollervolumeplugins_func.png)

#### CSIPersistentVolume Feature

![](./assets/img/startAttachDetachController_func.png)

![](./assets/img/probeattachablevolumeplugins_func.png)

### Controller Example 2: Wether start controller or not

类似可以控制，是否开启项目controller，去监听对应事件

- kubernetes/pkg/controller/certificates/rootcacertpublisher/publisher.go中定义了一个监听cm和ns的控制器。保证每个ns下都有一个名为kube-root-ca.crt的cm，保存访问api-server所需要certificates
- 使用这个功能需要开启feature-gate: BoundServiceAccountTokenVolume
- v1.13 alpha

![](./assets/img/startrootcapublisher_func.png)

### Controller Example 3：

![](./assets/img/startnodelifecyclecontroller_func.png)

### Kubectl Example:

![](./assets/img/unsecuredependencies_func.png)

![](./assets/img/probevolumeplugins_func.png)

## Howto

### Add known features

![kube_feature_default_feature_gates](./assets/img/kube_feature_default_feature_gates.png)

![](./assets/img/apiextensions_apiserver_default_feature_gates.png)

![](./assets/img/apiserver_default_feature_gates.png)

### Add flagset and flag.Value interface

![](./assets/img/apiserver_addflag.png)

![](./assets/img/feature_gate_set_method.png)

![image-20190726104158037](./assets/img/feature_gate_setfrommap_method.png)

![image-20190726104253443](./assets/img/feature_gate_string_and_type_method.png)

#### Cobra parse flags

![image-20190726110750052](./assets/img/cobra_Execute_and_Executec_method.png)

![image-20190726110525361](./assets/img/cobra_execure_method.png)

![image-20190726110348082](./assets/img/cobra_parseflags_method.png)

### DefaultMutableFeatureGate

![](./assets/img/default_mutable_feature_gate_struct.png)

![](./assets/img/new_feature_gate.png)

### Add from features

![](./assets/img/feature_gate_add_method.png)

### Check if feature enabled

![](./assets/img/feature_gate_enable_method.png)

- 启动时通过—feature-gates传入开启、关闭的featu、
- featureGate 对象有连个属性known，enabled
- 判定某个feature是否开启的时候，首先尝试从enabled判断，如果没有则从known获取默认值

## 疑问

PreRelease == Deprecated

PreRelease == GA