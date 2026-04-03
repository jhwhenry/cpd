# Offline Backup Runbook - 5.2.2

## Collect must-gather
[Collecting IBM Software Hub Must-gather & Health Check](https://www.ibm.com/support/pages/node/7261893)

## Clean up migration leftovers
### Clean up catalog-api migration leftovers
After you confirm that the projects, catalogs, and spaces are working as expected, run the following commands to clean up the migration resources:
Delete the pods associated with the migration:

```
oc delete pod \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```

Delete the jobs associated with the migration:

```
oc delete job \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```

Delete the config maps associated with the migration:
```
oc delete cm \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```

Delete the secrets associated with the migration:

```
oc delete secret \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```


Create volume snapshot for the cams-postgres-migration-pvc:

[Create volume snapshot](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.16/html/deploying_and_managing_openshift_data_foundation_using_red_hat_openstack_platform/volume-snapshots_osp)


After creating the volume snapshot, delete the persistent volume claim associated with the migration:

```
oc delete pvc cams-postgres-migration-pvc \
-n ${PROJECT_CPD_INST_OPERANDS}
```

### Cleaning up Db2 migration leftovers

- [Cleaning up migration resources](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrading-post-upgrade-setup-knowledge-catalog#ikc_post_upgrade__cleanup-mig-resources__title__1)

- [Cleaning up Db2U resources](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrading-post-upgrade-setup-knowledge-catalog#ikc_post_upgrade__cleanup-db2u__title__1)

- Remove the OLM artifacts for Db2 as a service.

```
cpd-cli manage delete-olm-artifacts \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=db2aaservice
```

Confirm that the db2aaservice operator resources were cleaned up by running these commands:
```
oc get sub -n${PROJECT_CPD_INST_OPERATORS} | grep db2aa

oc get csv -n${PROJECT_CPD_INST_OPERATORS} | grep db2aa

oc get catsrc -n${PROJECT_CPD_INST_OPERATORS} | grep db2aa

oc get deploy -n${PROJECT_CPD_INST_OPERATORS} | grep db2aa
```
 
## Updating cpd-br tenant service:

```
export OADP_OPERATOR_NS=oadp-operator
```

Make sure the cpdbr-tenant service is upgraded to the CPD 5.2.2 version
```
cpd-cli oadp install \
--component=cpdbr-tenant \
--cpdbr-hooks-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd \
--cpfs-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs \
--namespace=${OADP_OPERATOR_NS} \
--tenant-operator-namespace=${PROJECT_CPD_INST_OPERATORS} \
--cpd-scheduler-namespace=${PROJECT_SCHEDULING_SERVICE} \
--skip-recipes=true \
--upgrade=true \
--log-level=debug \
--verbose
```

## Force reconciliation and recreate B&R configmaps for all services

This script will save, and destroy these configmaps, and force reconcile all the services.
<br>
[cpdbr-service-refresh](https://github.com/IBM/cpd-cli/blob/master/cpdops/files/cpdbr-service-refresh.sh)

**Note:**
Download the script to local client work station before the OADP backup.
<br>

Run the script and force reconcile all the services.

```
./cpdbr-service-refresh.sh -t ${PROJECT_CPD_INST_OPERATORS}
```
