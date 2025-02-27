---
title: Extend Chaos Daemon Interface
---

import PickHelmVersion from '@site/src/components/PickHelmVersion'

In [Add new chaos experiment type](add-new-chaos-experiment-type.md), you have added HelloWorldChaos, which can print `Hello World!` in the logs of Chaos Controller Manager. To enable the HelloWorldChaos to inject some faults into the target Pod, you need to extend interface for Chaos Daemon.

:::note
It's recommended to read [Chaos Mesh architecture](architecture.md) before you go forward.
:::

This document covers:

- [Selector](#selector)
- [Implement the gRPC interface](#implement-the-grpc-interface)
- [Verify the experiment](#verify-the-experiment)

## Selector

In `api/v1alpha1/helloworldchaos_type.go`, you have defined `HelloWorldSpec`, which includes `ContainerSelector`:

```go
// HelloWorldChaosSpec is the content of the specification for a HelloWorldChaos
type HelloWorldChaosSpec struct {
    // ContainerSelector specifies target
    ContainerSelector `json:",inline"`

    // Duration represents the duration of the chaos action
    // +optional
    Duration *string `json:"duration,omitempty"`
}

...

// GetSelectorSpecs is a getter for selectors
func (obj *HelloWorldChaos) GetSelectorSpecs() map[string]interface{} {
    return map[string]interface{}{
        ".": &obj.Spec.ContainerSelector,
    }
}
```

In Chaos Mesh, Selector is used to define the scope of a chaos experiment, the target namespace, annotation, label, etc. Selector can also be some more specific values (for example, AWSSelector in AWSChaos). Usually, each chaos experiment requires only one Selector, with exceptions such as NetworkChaos because it sometimes needs two Selectors as two objects for network partition.

## Implement the gRPC interface

To allow Chaos Daemon to accept the requests from Chaos Controller Manager, you need to implement a new gRPC interface.

1. Add the RPC in `pkg/chaosdaemon/pb/chaosdaemon.proto`:

   ```proto
   service chaosDaemon {
      ...

      rpc ExecHelloWorldChaos(ExecHelloWorldRequest) returns (google.protobuf.Empty) {}
   }

   message ExecHelloWorldRequest {
      string container_id = 1;
   }
   ```

   You need to update the Golang code generated by this proto file:

   ```bash
   make proto
   ```

2. Implement gRPC services in Chaos Daemon.

   In the `pkg/chaosdaemon` directory, create a file named `helloworld_server.go` with the following contents:

   ```go
   package chaosdaemon

   import (
   "context"
   "fmt"

   "github.com/golang/protobuf/ptypes/empty"

   "github.com/chaos-mesh/chaos-mesh/pkg/bpm"
   pb "github.com/chaos-mesh/chaos-mesh/pkg/chaosdaemon/pb"
   )

   func (s *DaemonServer) ExecHelloWorldChaos(ctx context.Context, req *pb.ExecHelloWorldRequest) (*empty.Empty, error) {
   log.Info("ExecHelloWorldChaos", "request", req)

   pid, err := s.crClient.GetPidFromContainerID(ctx, req.ContainerId)
   if err != nil {
       return nil, err
   }

   cmd := bpm.DefaultProcessBuilder("sh", "-c", fmt.Sprintf("ps aux")).
       SetNS(pid, bpm.MountNS).
       SetContext(ctx).
       Build()
   out, err := cmd.Output()
   if err != nil {
       return nil, err
   }
   if len(out) != 0 {
       log.Info("cmd output", "output", string(out))
   }

   return &empty.Empty{}, nil
   }
   ```

   After `chaos-daemon` receives the `ExecHelloWorldChaos` request, you can see a list of processes in the current container.

3. Send a gRPC request when applying the chaos experiment.

   Each chaos experiment has its life cycle: `apply` and then `recover`. However, there are some chaos experiments that cannot be recovered by default (for example, PodKill in PodChaos, and HelloWorldChaos). These are called OneShot experiments. You can find `+chaos-mesh:oneshot=true` in the file that defines the schema type of chaos experiment type.

   Chaos Controller Manager needs to send a request to Chaos Daemon when HelloWorldChaos is in `recover`. To do this, you need to modify `controllers/chaosimpl/helloordchaos/types.go`:

   ```go
   package helloworldchaos

   import (
   "context"

   "github.com/chaos-mesh/chaos-mesh/api/v1alpha1"
   "github.com/chaos-mesh/chaos-mesh/controllers/chaosimpl/utils"
   "github.com/chaos-mesh/chaos-mesh/controllers/common"
   "github.com/chaos-mesh/chaos-mesh/pkg/chaosdaemon/pb"
   "github.com/go-logr/logr"
   "go.uber.org/fx"
   "sigs.k8s.io/controller-runtime/pkg/client"
   )

   type Impl struct {
   client.Client
   Log     logr.Logger
   decoder *utils.ContianerRecordDecoder
   }

   // This corresponds to the Apply phase of HelloWorldChaos. The execution of HelloWorldChaos will be triggered.
   func (impl *Impl) Apply(ctx context.Context, index int, records []*v1alpha1.Record, obj v1alpha1.InnerObject) (v1alpha1.Phase, error) {
   impl.Log.Info("Apply helloworld chaos")
   decodedContainer, err := impl.decoder.DecodeContainerRecord(ctx, records[index])
   if err != nil {
       return v1alpha1.NotInjected, err
   }
   pbClient := decodedContainer.PbClient
   containerId := decodedContainer.ContainerId

   _, err = pbClient.ExecHelloWorldChaos(ctx, &pb.ExecHelloWorldRequest{
       ContainerId: containerId,
   })
   if err != nil {
       return v1alpha1.NotInjected, err
   }

   return v1alpha1.Injected, nil
   }

   // This corresponds to the Recover phase of HelloWorldChaos. The reconciler will be triggered to recover the chaos action.
   func (impl *Impl) Recover(ctx context.Context, index int, records []*v1alpha1.Record, obj v1alpha1.InnerObject) (v1alpha1.Phase, error) {
   impl.Log.Info("Recover helloworld chaos")
   return v1alpha1.NotInjected, nil
   }

   func NewImpl(c client.Client, log logr.Logger, decoder *utils.ContianerRecordDecoder) *common.ChaosImplPair {
   return &common.ChaosImplPair{
       Name:   "helloworldchaos",
       Object: &v1alpha1.HelloWorldChaos{},
       Impl: &Impl{
           Client:  c,
           Log:     log.WithName("helloworldchaos"),
           decoder: decoder,
       },
       ObjectList: &v1alpha1.HelloWorldChaosList{},
   }
   }

   var Module = fx.Provide(
   fx.Annotated{
       Group:  "impl",
       Target: NewImpl,
   },
   )
   ```

   :::note
   In this chaos experiment, there is no need to recover the chaos action. This is because HelloWorldChaos is a OneShot experiment. For the chaos experiment type you developed, you can implement the logic of the recovery function as needed.
   :::

## Verify the experiment

To verify the experiment, perform the following steps.

1. Build the Docker image and push it to your local Registry. If the Kubernetes cluster is deployed using kind, you need to load the image to kind:

   ```bash
   make image
   make docker-push
   kind load docker-image localhost:5000/pingcap/chaos-mesh:latest
   kind load docker-image localhost:5000/pingcap/chaos-daemon:latest
   kind load docker-image localhost:5000/pingcap/chaos-dashboard:latest
   ```

2. Update Chaos Mesh:

   <PickHelmVersion className="language-bash">{`helm upgrade chaos-mesh helm/chaos-mesh --namespace=chaos-testing --version latest`}</PickHelmVersion>

3. Deploy the target Pod for testing. Skip this step if you have already deployed this Pod:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/chaos-mesh/apps/master/ping/busybox-statefulset.yaml
   ```

4. Create a new YAML file with the following content:

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: HelloWorldChaos
   metadata:
     name: busybox-helloworld-chaos
   spec:
     selector:
       namespaces:
         - busybox
     mode: all
     duration: 1h
   ```

5. Apply the chaos experiment:

   ```bash
   kubectl apply -f /path/to/helloworld.yaml
   ```

6. Verify the results. You can check several logs:

   - Check the logs of Chaos Controller Manager:

   ```bash
   kubectl logs chaos-controller-manager-{pod-post-fix} -n chaos-testing
   ```

   Example output:

   ```log
   2021-06-25T06:02:12.754Z        INFO    records apply chaos     {"id": "busybox/busybox-1/busybox"}
   2021-06-25T06:02:12.754Z        INFO    helloworldchaos Apply helloworld chaos
   ```

   - Check the logs of Chaos Daemon:

   ```bash
   kubectl logs chaos-daemon-{pod-post-fix} -n chaos-testing
   ```

   Example output:

   ```log
   2021-06-25T06:25:13.048Z        INFO    chaos-daemon-server     ExecHelloWorldChaos     {"request": "container_id:\"containerd://af1b99df3513c49c4cab4f12e468ed1d7a274fe53722bd883256d8f65bc9f681\""}
   2021-06-25T06:25:13.050Z        INFO    background-process-manager      build command   {"command": "/usr/local/bin/nsexec -m /proc/243383/ns/mnt -- sh -c ps aux"}
   2021-06-25T06:25:13.056Z        INFO    chaos-daemon-server     cmd output      {"output": "PID   USER     TIME  COMMAND\n    1 root      0:00 sleep 3600\n"}
   2021-06-25T06:25:13.070Z        INFO    chaos-daemon-server     ExecHelloWorldChaos     {"request": "container_id:\"containerd://88f6a469e5da82b48dc1190de07a2641b793df1f4e543a5958e448119d1bec11\""}
   2021-06-25T06:25:13.072Z        INFO    background-process-manager      build command   {"command": "/usr/local/bin/nsexec -m /proc/243479/ns/mnt -- sh -c ps aux"}
   2021-06-25T06:25:13.076Z        INFO    chaos-daemon-server     cmd output      {"output": "PID   USER     TIME  COMMAND\n    1 root      0:00 sleep 3600\n"}
   ```

   You can see `ps aux` in two separate lines, which are corresponding to two different Pods.

   :::note
   If your cluster has multiple nodes, you will find more than one Chaos Daemon Pod. Try to check logs of every Chaos Daemon Pods and find which Pod is being called.
   :::

## Next steps

If you encounter any problems in this process, create an [issue](https://github.com/pingcap/chaos-mesh/issues) in the Chaos Mesh repository.

If you are curious about how all of these come into effect, you can read the README files of different `controllers` in the `controller` directory to learn their functionalities. For example, [controllers/common/README.md](https://github.com/chaos-mesh/chaos-mesh/blob/master/controllers/common/README.md).

Now you are ready to become a Chaos Mesh developer! You are welcome to visit the [Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh) repository to find a [good first issue](https://github.com/chaos-mesh/chaos-mesh/labels/good%20first%20issue) and get started!
