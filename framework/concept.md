# Concept
Spiral framework has build as internal platform/tool to modularize development 
of various applications by improving re-usability to written code. Components
including Database, ORM, ODM and Stemplter has been build around the idea of writing
applications inside applications (by creating higher level abstractions for business logic)
by providing namespacing and isolation of every possible level of your application. 

Existed components are loosely coupled not only to each other, but hopefully 
to your application as well which should provide long term support and updates
without massing code rewriting.

Current implementation of core classes and components is intended to be kept light
and not overcompilcated to ensure that no projects are getting forced into specific
architecture (we simply don't know what is good architcture for your appliction can be)
and developer are given with enought room for experimentation and framework modification.
