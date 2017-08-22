The below is additional tips using FAQ format

 
# How does kubernetes work?

This is a 2 min crash course 

1. The system is a giant DSC (Desired State Configuration). Data is sent as strong typed data to *api-server* which validates and saves to *etcd*
2. A set of *control loops* that run inside *controller-manager*. Each compares *actual-state-of-the-world* to *desired-state* and applies the delta. 
3. Applying *desired-state* may entail changing *actual-state* such the case of *attaching disk to node* or writing back to *etcd* such as the case for scheduler (which runs a *control loop* as well) identifying a target node for pod. Or both such as the case of attempting to change *actual-state* and failing.
4. Some *state* changes spans multiple *control loops* and/or *control plane components* such as running a pod that requires volumes, Where *controller-manager* attaches the disk via *cloud provider* to node, then kubelet - which again, you guessed it right runs a *control loop* - on node performs format, mount on node. 
5. The entire system pulls on the *api-server* for state changes, *api-server* does not reach out to nodes except in case of you running *kubectl proxy ...*. As in heart beats are sent up from nodes to *api-server* not the other way around.
6. Key components:
  
  a. *api-server* sets on top of *etcd*

  b. *controler-manager* runs multiple *control loops* such as for nodes, volumes, RCs..

  c. *Kubelet* runs on nodes performs 1) runs containers via integrating with container runtime (yes, Kubernetes can do docker, RKT or soon CRI). 2) Collects node metadata (including health as heartbeats) and send it to *api-server*. Metadata is collected mainly via component called *CAdvisor* which abstracts the details of cloud specific to Kubernetes.

  d. *Proxy* runs -again as a control loop - on nodes to configure *iptables* to enable scenarios such ip-per-pod and Kubernetes services

7. Plugins, providers and all external-like components are called from various *control plane* components. Yes most of them run on master and nodes (not just master).

# What should i do before start coding?

1. Read https://github.com/kubernetes/community specifically contribution and developer guide.
2. Engage with the SIG that covers whatever you are about to contribute to https://github.com/kubernetes/community/blob/master/sig-list.md through your ideas at them get feedback.
3. Read the design doc for whatever component you will change/integrate with (now this is hard because they are scatterd all over the place). Search and ask on slack for pointers.
4. Break something, fix it. lather, rense & repeat.

> The community is a big fan of small PRs, bigger ones are harder, a lot harder to merge with upstream. 

# OMG, Debugging. What now?

Start with the following it is a DSC, repeat its a DSC. Meaning noting you call or do will be executed immediately. Hence logging+tracing is your best friend. Here are few tips:

1. All the *contol loops* run in a sand-box like env (simple example using go panic/recover https://blog.golang.org/defer-panic-and-recover). Runtime panics are trapped and stack-trace logged to stdout/stderr.
2. On debian systems, systemd is used to start *control plane* as containers using various units. Use ``` sudo systemctl status <<UNIT>> ``` and ``` sudo journalctl ``` to get more information if your containers refused to start.
3. Kubernetes uses glog library for logging. Make sure that you use correct the correct V ``` glog.V(<<LEVEL>>).XXXX ``` And make sure that you start your containers with the correct V parameters **acs-engine default to use --v=2** change to higher value. 
4. Use labels and selectors to place your pods on nodes you are already watching the logs for.
5. Use docker logs to see what is happening 

```
#on master, getting+following logs for controller
sudo docker ps | grep "hyperkube control" | awk '{print $1}' | xargs sudo docker logs -f

#on nodes, getting+following logs for kubelet 
sudo docker ps | grep "/hyperkube kubelet" | awk '{print $1}' | xargs sudo docker logs -f
```
6. Use tmux or screens to have multi window tracing different control plane components

# update-x, verify-x WTF are these?

Probably the most icky part of contribution is navigating and using the ./hack/ scripts. But there is a method to the madness. Instead of describing them, let talk- through some - why they are needed:

1. *api-server* exposes REST with JSON,Protobuff and YAML endpoint (yes it performs protocol negotiation for every request). Node->api-master uses protobuff only. For that reason everytime types are updated you will need to run update-generated-protobuf* family of scripts. 
2. Kubernetes does not use the default golang JSON libs, instead it uses a custom library that generates objects graph as code (instead of generating at runtime and maintain LRU cache). For that reason everytime types are updated you will need to run update-codegen* family of scripts. 
3. All the api/ref/swagger docs are generated via updated-*-docs. 
4. Bazel build files (this BUILD files in each directory) has thier own update-bazel script, You will need to verify that your test packages are in Bazel files. 
5. gofmt, golint, goevt have thier own scripts. 
6. All none code data are included in code - as base64 strings - via generate-bindata script.
7. The various runtime pieces are generated including api repo, client (used by kubectl and *control plane* components) are generated  via *-runtime,*-codegen scripts. 
8. Each update-* script has a counter verify-* script. 

# Can I develop without Azure clusters? 

Why do you want to do that? -- in Theory (I didn't try that e2e) yes you can mock the ambient information needed for Azure provider to run correctly (mainly metadata endpoint, disable MSI). You can use ./hack/dev-build-up.sh to run a small env on your dev box. Don't do that. 

# What if i want to run old code and new code on the same cluster?

for whatever reason you needed that - welcome to the dark side -. You can use Agent pools for this. Create a cluster with multiple agent pools, each agent pool can run using a different version of kubelet. *They will all use one version of control plane components*. This will allow you to compare old and new on the same cluster (Just in casse, This is not a production scenario, don't do that on production)

> This assumes that your types didn't change.

# Dev workflow? 

if you are doing one component at a time then fork, branch, merge to your master then PR from your master is best approach. However if you are doing many components at a time you may want to consider having many source trees and bind mount them at go/src/k8s 

# Any advise on dev tool chain? 

That is quite a personal question, whatever work for you is the right one. But just in case you are looking for ideas. Here is my dev-boxs common configuration:
1. tmux 
2. vim + vim-go (https://github.com/fatih/vim-go) + ctags
3. Other things i have on vim are nerdtree, syntastic, tagbar, YouCompleteMe, lightline, and ctrlp
