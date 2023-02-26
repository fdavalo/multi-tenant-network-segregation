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

 * example of front load balancer and VIP by tenant (here using haproxy)

            #---------------------------------------------------------------------
            # main frontend which proxys to the backends
            #---------------------------------------------------------------------
            frontend main-1085
                bind 10.6.85.10:80
                acl url_static       path_beg       -i /static /images /javascript /stylesheets
                acl url_static       path_end       -i .jpg .gif .png .css .js

                use_backend static          if url_static
                default_backend             app-1085

            frontend main-1081
                bind 10.6.81.10:80
                acl url_static       path_beg       -i /static /images /javascript /stylesheets
                acl url_static       path_end       -i .jpg .gif .png .css .js

                use_backend static          if url_static
                default_backend             app-1081

            #---------------------------------------------------------------------
            # round robin balancing between the various backends
            #---------------------------------------------------------------------
            backend app-1085
                balance     roundrobin
                server  app1 10.6.85.196:80 check
                server  app2 10.6.85.197:80 check

            backend app-1081
                balance     roundrobin
                server  app1 10.6.81.196:80 check
                server  app2 10.6.81.197:80 check

 * example of wildcard DNS setting for each tenant (here using named)

            *.tenant1085.ocp1 IN A 10.6.85.10
            *.tenant1081.ocp1 IN A 10.6.81.10

 
