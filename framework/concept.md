# Concept
Spiral framework has build as internal platform/tool to modularize development 
of various projects by improving re-usability to written code. Components,
including Database, ORM, ODM and Stemplter has been designed around the idea of writing
applications inside applications (creating higher level abstractions for business/domain logic)
by providing namespace and isolation support for as much levels of your application as possible.

Existed components are loosely coupled not only to each other, but hopefully 
to your application as well, which should, potentially, provide longer support 
and updates without massing code rewriting.

Current implementation of core classes and components is intended to be kept light
and not overcompilcated to ensure that no projects are getting forced into specific
architecture (i simply don't know what is good architecture for your application can be)
so developer are given enought room for experimentation and framework modification.

If you have an idea how to improve code reusability or unifify framework interfaces be
free to propose your ideas on a [forum](https://groups.google.com/forum/#!forum/spiral-framework).
