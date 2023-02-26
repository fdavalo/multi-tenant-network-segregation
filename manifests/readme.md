## Notes

in default ingresscontroller, add

            namespaceSelector:
              matchExpressions:
                - key: tenant
                  operator: NotIn
                  values:
                    - tenant1085
                    - tenant1081

to remove tenants routes from default router
