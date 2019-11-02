# Security - Role Based Access Control
The framework includes the component `spiral/security` which provides the ability to authorize user/actor access to the
specific resources and actions based on list of associated privileges. The components implements "Flat RBAC" patterns
described in [NIST RBAC research](https://csrc.nist.gov/projects/role-based-access-control). 

Implementation includes multiple additions such as:
- an additional layer of *rules* which provides the ability to control the privilege based on 
resource context (for example "only admins and post authors can edit target post").
- the ability to assign the role to multiple privileges using wildcard pattern
- the ability to overwrite the role to permission assignment using higher priority rule

Such additions make possible to use the component as the framework for ACL, DAC and ABAC security models.

> The component is enabled in Web and GRPC bundles by default.

Make sure to enable the `Spiral\Bootloader\Security\GuardBootloader` to activate the component, no configuration is 
required.

// keep writing
