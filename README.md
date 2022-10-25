# 利用EKS Blueprint for Terraform快速部署集群

在本次实验中，我们会使用EKS Blueprint for Terraform快速部署一个集群，在上面部署Karpenter和样例应用，并利用Karpenter进行节点的弹性伸缩。

## 部署Cloud9
Cloud9是AWS提供的基于EC2的在线IDE，后续的所有操作都会在这个IDE上完成。

- 进入AWS控制台，在搜索框输入`Cloud9`，进入Cloud9控制台
- 点击`Create Environment`
- 为新的环境任取一个名字，输入`IDE`并点击`Next Step`
- 在`Instance Type`中，选择`t3.small`，其他保持默认，点击`Next Step`
- 点击`Create Environment`，新的环境会自动弹出。
- 点击标签栏右上角的绿色加号，选择`New Terminal`，启动一个终端。

## 初始化
我们提供了初始化脚本以赋予Cloud9权限，并安装一些需要的依赖项。运行如下命令：
```bash
git clone https://github.com/yubingjiaocn/eksblueprint-tf-workshop
cd eksblueprint-tf-workshop
./init.sh
```
在脚本中，我们会安装`kubectl`, `eksctl`, `helm`客户端，并安装了`Terraform`。

## 运行
所需的Terraform模板已放在Repo中，运行以下命令以进行初始化：
```bash
terraform init
```
该命令会查找所有需要的依赖Module，并将其下载到本地。初始化完成后，即可进行部署。部署前，建议运行以下命令以预览更改：
```bash
terraform plan
```
预览无误后，正式进行部署：
```bash
terraform apply -auto-approve
```
部署大概需要15-20分钟。

## 访问刚刚创建的集群
部署完成后，在命令的输出中可以看到类似于`configure_kubectl = "aws eks --region ap-southeast-1 update-kubeconfig --name eksblueprint-tf-workshop"`的命令。

运行该命令，即可使用`kubectl`访问集群。

您可以运行`kubectl get node`以验证是否能正常访问集群。

## 安装Karpenter
EKS Blueprint提供了一些常见组件，可作为Add-on部署到集群中。在本次实验中，我们采用Add-On方式安装Karpenter。

将下面内容加入到`main.tf`中：

```terraform
module "eks_blueprints_kubernetes_addons" {
  source = "github.com/aws-ia/terraform-aws-eks-blueprints//modules/kubernetes-addons?ref=v4.12.2"

  eks_cluster_id = module.eks_blueprints.eks_cluster_id

  #K8s Add-ons
  enable_aws_load_balancer_controller = true
  enable_karpenter                    = true
  enable_metrics_server               = true
}
```

Karpenter需要`Provisioner`才能创建集群。Provider指定了新创建节点的实例类型，容量模式，子网等设置。在`data.tf`加入以下内容:

```terraform
data "kubectl_path_documents" "karpenter_provisioners" {
  pattern = "${path.module}/kubernetes/karpenter/*"
  vars = {
    azs                     = join(",", local.azs)
    iam-instance-profile-id = "${local.name}-${local.node_group_name}"
    eks-cluster-id          = local.name
    eks-vpc_name            = local.name
  }
}
```

在`main.tf`里加入以下内容：
```terraform
resource "kubectl_manifest" "karpenter_provisioner" {
  for_each  = toset(data.kubectl_path_documents.karpenter_provisioners.documents)
  yaml_body = each.value

  depends_on = [module.eks_blueprints_kubernetes_addons]
}
```

由于我们增加了新的Module，需要重新初始化Terraform。但不必担心，状态仍会保留在Terraform中。运行以下命令：
```bash
terraform init
```
初始化完成后，即可进行部署。
```
terraform plan
terraform apply -auto-approve
```
部署完成后，使用下列命令检查资源是否被正确创建：
```bash
kubectl get provisioner
kubectl get pod -n karpenter
```
## 用Karpenter进行弹性伸缩
运行以下命令创建一个资源消耗为1C/1.5G的示例Deployment:
```
kubectl apply -f kubernetes/inflate/deployment.yaml
```
我们虽然创建了一个Deployment，但是未创建任何Pod。我们可以利用`kubectl scale`命令调整Pod数量，观察节点变化：

```bash
kubectl scale deployment inflate --replicas=1
```

此时运行如下命令，即可看到节点数量变化：
```bash
kubectl get nodes -L karpenter.sh/capacity-type -L topology.kubernetes.io/zone -L karpenter.sh/provisioner-name -L node.kubernetes.io/instance-type
```