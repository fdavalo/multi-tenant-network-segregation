## Notes

 * in default ingresscontroller, add

            namespaceSelector:
              matchExpressions:
                - key: tenant
                  operator: NotIn
                  values:
                    - tenant1085
                    - tenant1081

   to remove tenants routes from default router
   
 * add services for each ingress tenant before adding the ingresscontrollers for each
   * you wont have to patch the generated service
   * you will avoid nodeports from generated service otherwise, even the service patched, they will stay   
