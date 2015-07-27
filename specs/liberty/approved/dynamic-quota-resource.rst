..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================
Provide support for creating dynamic quota resources 
====================================================

https://blueprints.launchpad.net/nova/+spec/dynamic-quota-resource

Problem description
===================

* During creation or deletion of an instance, quota needs to be calculated and
  committed accordingly.
* Currently, quota calculations are based on 14 static resources hardcoded in
  the nova/quota.py.
* These static resources are cores, ram, disk, instances and so on.

* There is no support provided for an openstack admin to create dynamic quota
  resources and assign quota for a specific project, based on the newly created
  dynamic resource.
* This is in relation to usecases, where an openstack admin wants to allow a
  specific project to create only 5 instances in a particular availability
  zone, particular flavor type, hardware type (SSD) or any random category that
  an admin chooses.
* Quota calculation during instance creation and deletion should be based on
  all quota resources - static and dynamic.

**How an admin chooses to make the selection of hardware is out of scope of
this spec.**

Use Cases
---------

**Actors**

* Alice - Cloud Admin
* Bob - End User

Alice is a Cloud Admin who is responsible for assigning quota to onboarding
tenants.  
Bob is a end user who will be able to create instances based on the quota
assigned to him.

1. Alice wants to assign quota to different tenants based on standard static
   quota resources like Cores, RAM, disk etc. But, in addition to static
   quota resources, Alice also wants to track total number of instances being
   created based on following use cases. 
  
  a. Alice has a special type of HW (containing SSDs for example) and want to
     restrict total number of instances being created using that hardware.
  b. Alice has a created a special flavor corresponding to a specific aggregate
     for testing purposes and want to restrict total number of instances being
     created using that flavor.
  c. There is a special availability zone and Alice wants to restrict total
     number of instances being created in that AZ.

2. If Alice has assigned quota based on dynamic quota resources for Bob, he
   will want to pass a     quota resource against which his quota should be
   considered.


Project Priority
-----------------

None


Proposed change
===============

1. In order to assign quota for dynamic quota resources, os-quota-sets wsgi
extension needs to be updated to accept dynamic quota resource key and its
associated value. It could be sent as individual key-value pairs in the
quota-set dictionary. 
 
 a. Currently nova quota-update calls os-quota-sets wsgi extension with
following payload
  
  * -d '{"quota_set": {"tenant_id": "7c0d996ce4e14d86aca38878eb765a68",
    "cores": "12" }}'  

 b. It would be great if cloud admin could name the dynamic quota resource with
    whatever name he wants. We will not do any validation on the name of the
    quota resource. We will only validate if the type of value is integer. So, 
    we will need to update the the extension to accept arbitrary key value 
    pairs. For example -

  * -d '{"quota_set": {"tenant_id": "7c0d996ce4e14d86aca38878eb765a68",
    "cores": "12", "ssd_hw": "5", "flavor_xyz": "1", "special_az": "5" }}' 

2. Currently, static quota resources are stored in quotas table as::
 
    *************************** 1. row ***************************
    id: 1
    created_at: 2015-07-15 02:47:32
    updated_at: NULL
    deleted_at: 2015-07-21 04:26:24
    project_id: 7c0d996ce4e14d86aca38878eb765a68
    resource: cores
    hard_limit: 12
    deleted: 0

3. Now, dynamic quota resoruces will also be stored along with static quota
   resources in the table. For example::

    *************************** 1. row ***************************
    id: 1
    created_at: 2015-07-15 02:47:32
    updated_at: NULL
    deleted_at: 2015-07-21 04:26:24
    project_id: 7c0d996ce4e14d86aca38878eb765a68
    resource: cores
    hard_limit: 12
    deleted: 0
    *************************** 2. row ***************************
    id: 2
    created_at: 2015-07-15 02:47:32
    updated_at: NULL
    deleted_at: 2015-07-21 04:26:24
    project_id: 7c0d996ce4e14d86aca38878eb765a68
    resource: ssd_hw
    hard_limit: 5
    deleted: 0
    *************************** 3. row ***************************
    id: 3
    created_at: 2015-07-15 02:47:32
    updated_at: NULL
    deleted_at: 2015-07-21 04:26:24
    project_id: 7c0d996ce4e14d86aca38878eb765a68
    resource: flavor_xyz
    hard_limit: 1
    deleted: 0
    *************************** 4. row ***************************
    id: 4
    created_at: 2015-07-15 02:47:32
    updated_at: NULL
    deleted_at: 2015-07-21 04:26:24
    project_id: 7c0d996ce4e14d86aca38878eb765a68
    resource: special_az
    hard_limit: 5
    deleted: 0

4. We will also track dynamic quota resources in a separate
   dynamic_quota_resource table. For example::

    *************************** 1. row ***************************
    id: 1
    resource: ssd_hw
    deleted: 0
    *************************** 2. row ***************************
    id: 2
    resource: flavor_xyz
    deleted: 0
    *************************** 3. row ***************************
    id: 3
    resource: special_az
    deleted: 0
 

5. When user does a nova quota-show or uses the API, he will get information on
   the dynamic quota resources for which his project has been assigned quota
   for. For example::
    +--------------+-------+    
    | Quota        | Limit |
    +--------------+-------+
    | flavor_xyz   | 1     |
    +--------------+-------+
    | ssd_hw       | 5     |
    +--------------+-------+
    | special_az   | 5     |
    +--------------+-------+
    | cores        | 12    |
    +--------------+-------+

5. How the dynamic quota resource name is derived during instance creation, is
   something we will need some discussion on. For now I propose the following:

   * Since, there will be multiple dynamic quota resources per project, we need
     to get an input from the user as to against which dynamic quota resource,
     should his request be tracked. This input could also be used in one of the
     hardware selection scheduler filter. (How filter will use this information
     is out of scope of this spec). We will throw an exception if a dynamic 
     quota resource is assigned for a project and the user has not specified one.

      * nova boot --flavor <flavor> --image <image> --dynamic_quota_resource
        <dynamic quota resource>

6. Once dynamic quota resource name is obtained, it will be used while creating
   quota reservations. Value of the dynamic quota resource will be decremented
   by 1. Also, we will store the resource-id of the dynamic quota resource 
   during instance creation. This will help us during instance deletion and we
   will be able to increment quota value of appropriate dynamic quota resource
   associated with the instance.

7. For all quota calculations, all the static resources are hard-coded and the
   resource dictionary is formed at the time of service initialization. So,
   multiple api workers form the same resource dictionary. With quota resources
   being created dynamically, we will have to query the DB
   (dynamic_quota_resources table) before every quota operation, to get the
   latest resource dictionary.  

Alternatives
------------

None

Data model impact
-----------------

* Create a new table dynamic_quota_resources with following spec::

    CREATE TABLE `dynamic_quota_resource` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `resource` varchar(255) NOT NULL,
      `deleted` int(11) DEFAULT NULL,
      PRIMARY KEY (`id`))

* Create a new column called quota_resource_id in instances table.

REST API impact
---------------

* Server create api needs to be updated to accept dynamic_quota_resource
  parameter.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:

Other contributors:

Work Items
----------

1. os-quota-sets extension needs to be updated to allow creation of dynamic
   quota resources.

2. DB scripts needs to be added to create dynamic_quota_resources table. Also,
   new column called 'quota_resource' needs to be added to instances table.

3. Server create api needs to be updated to accept dynamic_quota_resource
   parameter during instance creation.

4. QuotaEngine and DBQuotaDriver needs to be updated to account for dynamic
   quota resources during quota calculations.

Dependencies
============

None

Testing
=======

1. Apart from unit tests, functional tests will be added related to items 
   below.
    1. test creation of dynamic quota resource.
    2. show dynamic quota resources during os-quota-sets api call.
    3. increment/decrement dynamic quota resource value during
       creation/deletion of instance using dynamic quota resource.

Documentation Impact
====================

* Documentation will have to be updated to reflect creation of dynamic quota
  resource for cloud-admins. 
* Also, documentation will have to be updated to reflect new
  dynamic_quota_resource parameter to be passed during instance creation.

References
==========

None
