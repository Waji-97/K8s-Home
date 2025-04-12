# Image Uploading using Virtctl
Using Datavolume CR from CDI works good with Linux VMs. However, in the case of Window ISOs, we need to use the cdi-uploadproxy as below.

Port forward the cdi-uploadproxy service (need to be root)
```bash
kubectl port-forward -n kubevirt service/cdi-uploadproxy 18443:443 --kubeconfig=/home/waji/.kube/config
Forwarding from 127.0.0.1:18443 -> 8443
Forwarding from [::1]:18443 -> 8443
Handling connection for 18443
Handling connection for 18443
```

Upload the ISO 
```bash
virtctl image-upload --pvc-name=isohd --namespace=kubevirt --size=10Gi --insecure --image-path=/home/waji/Downloads/Win10_22H2_Korean_x64v1.iso --uploadproxy-url=https://127.0.0.1:18443
Flag --pvc-name has been deprecated, specify the name as the second argument instead.
PVC kubevirt/isohd not found 
PersistentVolumeClaim kubevirt/isohd created
Waiting for PVC isohd upload pod to be ready...
Pod now ready
Uploading data to https://127.0.0.1:18443

5.42 GiB / 5.42 GiB [--------------------------------------------------------------------------------------------------------------------------] 100.00% 1.52 MiB p/s 1h0m58s

Uploading data completed successfully, waiting for processing to complete, you can hit ctrl-c without interrupting the progress
Processing completed successfully
Uploading /home/waji/Downloads/Win10_22H2_Korean_x64v1.iso completed successfully
```

Now we can deploy a VirtualMachine CR for Windows VM that uses the above volume.