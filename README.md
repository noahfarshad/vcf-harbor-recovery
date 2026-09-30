# Regional Harbor stuck mid-removal

A Supervisor region teardown can remove Harbor without finishing the uninstall. The name stays claimed by a cluster-scoped `ClusterDomainResolutionEntry`. A role binding in `kube-system` does not reach it, so the UI retry does nothing. Image pulls fail closed, and VKS on that Supervisor goes down with the registry.

Regional Harbor is the registry VCF 9.1 documents for Supervisor services, VCF CLI plugins, and VKS add-ons. A Harbor you install by hand on one Supervisor does not stand in for it.

https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/using-harbor-as-vcf-service.html

## Order

1. From the Supervisor, confirm `depot.kube-system.svc` is still present:

   ```bash
   kubectl get clusterdomainresolutionentries -A
   kubectl auth can-i delete clusterdomainresolutionentries.containerregistry.vmware.com
   ```

2. If that answer is no, apply `temp-elevated-cleanup.yaml`. It is temporary.

3. Clear the finalizer on that one object, and confirm the list is empty:

   ```bash
   kubectl patch clusterdomainresolutionentry depot.kube-system.svc --type=json \
     -p='[{"op":"remove","path":"/metadata/finalizers"}]'
   kubectl get clusterdomainresolutionentries -A
   ```

   Do this only after you have confirmed the object is the stuck depot entry. It is not a general cleanup.

4. Delete the cleanup binding. Install Harbor from Service Management, Available, Harbor, Install. If the install cannot create the depot service and endpoints, apply `temp-elevated-install.yaml` and let the reconcile finish. In the recovery this was written from, that reconcile took about 15 minutes. The TKG package came back with the depot.

5. Remove every temporary grant:

   ```bash
   kubectl delete clusterrolebinding temp-elevated-binding
   kubectl delete clusterrole temp-elevated
   ```

Do not leave delete rights on the registry API.
