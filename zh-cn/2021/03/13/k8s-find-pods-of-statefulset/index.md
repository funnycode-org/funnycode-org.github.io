# 怒喷k8s:竟然还要这么才能正确找到statefulset的pods





<!--more-->

## 一、前言
今天在排查一个线上的中间件集群，该中间件集群是通过 `helm` 部署到`k8s`集群当中，有一个`statefulset`总是有一个`pod`还没有`ready`，故想去看看为啥一直不正常。在k8s当中关联对象之间的关系，我想当然的使用`statefulset`的`label selector`去找，结果却找到了其他类似statefulset所调度的pod。问了一些人，有说使用selector都能找到，众所周知，在k8s的很多描述文件中都是使用label相互之间关联，但是为啥这次不行了呢？有说直接使用名称匹配即可，感觉不适合靠谱。具体怎么找呢？

## 二、问题复现
我的`statefulest`描述文件-`nginx1.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx1
  labels:
    app: nginx
spec:
  ports:
  - port: 80
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx1
spec:
  serviceName: "nginx1"
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```
第二个`statefulest`描述文件-`nginx2`.yaml:
```yaml:
apiVersion: v1
kind: Service
metadata:
  name: nginx2
  labels:
    app: nginx
spec:
  ports:
  - port: 80
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx2
spec:
  serviceName: "nginx2"
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```
可以看到2个`statefulset`的selector都是`app: nginx`. \
执行`kubectl get sts`:
```shell
NAME     READY   AGE
nginx1   1/1     5m16s
nginx2   1/1     2m19s
```
执行`kubectl get pods`:
```shell
NAME       READY   STATUS    RESTARTS   AGE
nginx1-0   1/1     Running   0          6m24s
nginx2-0   1/1     Running   0          3m27s
```
发现`statefulset`已经正常起来了。

此时发现使用label的selector已经不能正常得到s`tatefulset`所管理的`pod`，执行`kubectl get pods -l app=nginx`结果如下：
```shell
NAME       READY   STATUS    RESTARTS   AGE
nginx1-0   1/1     Running   0          9m30s
nginx2-0   1/1     Running   0          6m33s
```
所以怎么才能正确得到`statefulset`的`pod`呢？
### 源码解析
源码根据[`kubernets`](https://github.com/kubernetes/kubernetes/tree/v1.20.2)源码`v1.20.2`版本。\
可以大概猜到代码应该在`pkg`下的`statefulest`的`controller`，定位到`pkg/controller/statefulset/stateful_set.go`,稍稍搜索该代码会发现第一处代码：
```go
// getPodsForStatefulSet returns the Pods that a given StatefulSet should manage.
// It also reconciles ControllerRef by adopting/orphaning.
//
// NOTE: Returned Pods are pointers to objects from the cache.
//       If you need to modify one, you need to copy it first.
func (ssc *StatefulSetController) getPodsForStatefulSet(set *apps.StatefulSet, selector labels.Selector) ([]*v1.Pod, error) {
	// List all pods to include the pods that don't match the selector anymore but
	// has a ControllerRef pointing to this StatefulSet.
	pods, err := ssc.podLister.Pods(set.Namespace).List(labels.Everything())
	if err != nil {
		return nil, err
	}

	filter := func(pod *v1.Pod) bool {
		// Only claim if it matches our StatefulSet name. Otherwise release/ignore.
		return isMemberOf(set, pod)
	}

	cm := controller.NewPodControllerRefManager(ssc.podControl, set, selector, controllerKind, ssc.canAdoptFunc(set))
	return cm.ClaimPods(pods, filter)
}
```
该处代码是获取`pod`被哪些`statefulset`所管理，上面我的2个`statefulset`都是被`selector``app: nginx`搜管理，所以根据`nginx1-0`和`nginx2-0`任一都会找到2个`satefulst`：`nginx1`和`nginx2`。

第二处代码是根据`statefulset`找到所管理的`pod`:
```go
// getPodsForStatefulSet returns the Pods that a given StatefulSet should manage.
// It also reconciles ControllerRef by adopting/orphaning.
//
// NOTE: Returned Pods are pointers to objects from the cache.
//       If you need to modify one, you need to copy it first.
func (ssc *StatefulSetController) getPodsForStatefulSet(set *apps.StatefulSet, selector labels.Selector) ([]*v1.Pod, error) {
	// List all pods to include the pods that don't match the selector anymore but
	// has a ControllerRef pointing to this StatefulSet.
	pods, err := ssc.podLister.Pods(set.Namespace).List(labels.Everything())
	if err != nil {
		return nil, err
	}

	filter := func(pod *v1.Pod) bool {
		// Only claim if it matches our StatefulSet name. Otherwise release/ignore.
		return isMemberOf(set, pod)
	}

	cm := controller.NewPodControllerRefManager(ssc.podControl, set, selector, controllerKind, ssc.canAdoptFunc(set))
	return cm.ClaimPods(pods, filter)
}
```
此时注意2处代码`filter`和`ClaimPods`方法。先看`filter`方法逻辑`isMemberOf`，该方法就会过滤找出`statefulset`所管理的`pod`列表。
```go
// isMemberOf tests if pod is a member of set.
func isMemberOf(set *apps.StatefulSet, pod *v1.Pod) bool {
	return getParentName(pod) == set.Name
}
```
明显该代码是匹配了`statefulset`的名字
```go
// statefulPodRegex is a regular expression that extracts the parent StatefulSet and ordinal from the Name of a Pod
var statefulPodRegex = regexp.MustCompile("(.*)-([0-9]+)$")

// getParentNameAndOrdinal gets the name of pod's parent StatefulSet and pod's ordinal as extracted from its Name. If
// the Pod was not created by a StatefulSet, its parent is considered to be empty string, and its ordinal is considered
// to be -1.
func getParentNameAndOrdinal(pod *v1.Pod) (string, int) {
	parent := ""
	ordinal := -1
	subMatches := statefulPodRegex.FindStringSubmatch(pod.Name)
	if len(subMatches) < 3 {
		return parent, ordinal
	}
	parent = subMatches[1]
	if i, err := strconv.ParseInt(subMatches[2], 10, 32); err == nil {
		ordinal = int(i)
	}
	return parent, ordinal
}
```
正则表达式很明显看出是根据`pod`的名找到`statefulset`的名字，这里根据`nginx1-0`就会找到`statefulset``nginx1`。

继续回到方法`ClaimPods`方法：
```go
// ClaimPods tries to take ownership of a list of Pods.
//
// It will reconcile the following:
//   * Adopt orphans if the selector matches.
//   * Release owned objects if the selector no longer matches.
//
// Optional: If one or more filters are specified, a Pod will only be claimed if
// all filters return true.
//
// A non-nil error is returned if some form of reconciliation was attempted and
// failed. Usually, controllers should try again later in case reconciliation
// is still needed.
//
// If the error is nil, either the reconciliation succeeded, or no
// reconciliation was necessary. The list of Pods that you now own is returned.
func (m *PodControllerRefManager) ClaimPods(pods []*v1.Pod, filters ...func(*v1.Pod) bool) ([]*v1.Pod, error) {
	var claimed []*v1.Pod
	var errlist []error

	match := func(obj metav1.Object) bool {
		pod := obj.(*v1.Pod)
		// Check selector first so filters only run on potentially matching Pods.
		if !m.Selector.Matches(labels.Set(pod.Labels)) {
			return false
		}
		for _, filter := range filters {
			if !filter(pod) {
				return false
			}
		}
		return true
	}
  ...

	for _, pod := range pods {
		ok, err := m.ClaimObject(pod, match, adopt, release)
    ...
	}
	return claimed, utilerrors.NewAggregate(errlist)
}
```
`match`方法已经揭露完整真相：现根据`selector`匹配，然后再`filter`里面的名称匹配。

## 三、总结

在`statefulset`中既使用了标签匹配，又使用了根据`pod`的名称截取出来`statefulset`的名称匹配。所以上面我的示例正确获取方法如下:
```shell
kubectl get pods -l app=nginx | grep -v NAME | awk '{print $1}' | grep -E --color '^(nginx1)-([0-9]+)$'
```
结果如下:
```shell
nginx1-0
```

