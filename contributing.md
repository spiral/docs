# Contributing
Feel free to contribute to framework or components development. I will glaldy accept your pull requests if it does match provided requiments and do not adds extra complexity to framework (see framework [concept](/framework/concept.md)).

## Requiments
* PSR-2 and PSR-4, soft width 100
* Every method must have doc comment describing return value and parameters, see existed style
* Test coverage of new functionality

## Help Needed In
* Test coverage of existed functionality (slowly doing by myself, currenly relying on [acceptance/compilation tests](https://travis-ci.org/spiral/application))
* Functionality proposals
* Code quality feedback and proposals (i bet there is decent amount of mistakes)
* Advices and critics
* Automatic splitting for components [repository](https://github.com/spiral/components)
* Grammar and spelling fixes :(
* Flexibility proposals (so far there almost no events in framework and at this point i'm too afraid to add any)
* Something important missing in this guide? Let me know! 

## Guide Improvements
If you feel like some "sugar" functionality (like shared bindings, constructor saturation or autowiring) can cause potential issues in future or reduce code testability please let me know so i can update guide and mention about potential architecture decigions or workarounds.

> If you found any issue which is better to be covered in documentation please open related issue. Since i'm not native english speaker feel free to create pull requests for any typo also.

## Critial Issues
If you found something which should't be there or bug which opens a security hole please let me know immediately by email wolfy.jd@gmail.com
