---
title: Symfony Serializer. Circular References vs Duplicate references. 
---

[API Platform](https:/api-platform.com) does some nice stuff with circular references in the serializer - it keeps track of all of the resources being serialized and when a resource that has already been seen on a branch is found it is replaced by the IRI for that resource.

This works really well... for a while... but when there are Many to Many relationships you can end up with a lot of duplicated resources in the serialized representations. 

