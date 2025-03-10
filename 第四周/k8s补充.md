# k8s补充

## PV & PVC & storageClass

PV由管理员创建，包含存储容量、访问权限等声明

PVC由用户创建，需要匹配PV的相关设置，然后需要找到匹配的PV进行绑定，绑定之后就可以在pod使用PVC（否则等待到创建相应的PV）

storageClass是相当于对PV的一个模板，帮助自动化创建PV的。