# Contributing
Feel free to contribute to framework or components development. I will glaldy accept your pull requests if it does match provided requiments and do not adds extra complexity to framework.

## Requiments
* PSR-1, PSR-2, PSR-4, line width 100
* Every method must have doc comment describing return value and parameters
* No design and concept violations (i love crazy ideas, but please let's talk first :))
* Test coverage of new functionality
* Defaul application configuration and tests must not fail ([example](https://travis-ci.org/spiral/application), failing on HHVM :/)

## Help Needed In
* Test coverage of existed functionality
* Functionality proposals
* Questionable code improvements and lookup
* Critics and Improvement proposals
* Organizational questions
* Automatic splitting for components [repository](https://github.com/spiral/components)
* Flexibility proposals (so far there almost no events in framework and at this point i'm too afraid to add any)
* Something important missing in this guide? Let me know! 

## Guide Improvements
If you feel like some "sugar" functionality (like shared bindings, constructor saturation or autowiring) can cause potential issues in future or reduce code testability please let me know so i can update guide and mention about potential architecture decigions or workarounds.

> If you found any issue which is better to be covered in documentation please open related issue. Since i'm not native english speaker feel free to create pull requests for any typo also.

## Critial Issues
If you found something which should't be there or bug which opens a security hole please let me know immediately by email wolfy.jd@gmail.com
