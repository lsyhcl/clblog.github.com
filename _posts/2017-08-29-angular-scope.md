---
layout: post
title:  "angularJS的作用域"
date:   2017-08-31 10:58:05
categories: angularJS
excerpt: angularJS的作用域的学习
---

* content
{:toc}

## $socpe和作用域(scope)
1、AngularJS中的$scope对象是模板的域模型，也称为作用域实例。通过对其属性赋值，可以传递数据到模板（页面）进行渲染。

2、作用域(scope)可以加入域模板相关的数据和提供相关的功能。

html
    
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Title</title>
        <script src="js/angular.js"></script>
    </head>
    
    <body ng-app="a">
    <div ng-controller="HelloCtrl">
        <label>Say hello to :
            <input type="text" ng-model="name">
        </label>
        <h1>Hello, {{getName()}}!</h1>
    </div>
    </body>
    
    
    <script language="JavaScript" type="text/javascript">
        var moA = angular.module("a", []);
        moA.controller("HelloCtrl", function ($scope) {
            $scope.name = 'World';
            $scope.getName = function () {
                return $scope.name;
            }
        });
    
        moA.controller("")
    </script>
    
    </html>

## ng-controller
1、每个$scope都是scope的实例。scope拥有很多方法，用于控制作用域的生命周期、提供事件传播(event-propagation)功能，以及支持模板的渲染等。

2、ng-controller指令会调用scope对象的$new()方法创建新的作用域实例($scope)。例如上个例子中的代码。

3、新创建的作用域实例$scope会拥有$parent属性，并指向他的父作用域

## ng-repeat
1、某些指令会创建子作用域。例如ng-repeat。下面例子中的&lt;li&gt;标签下的counry.population，就在新生成的作用域(scope)中

2、下面的函数worldsPercentage实际是在WorldCtrl的作用域里的，但是repeat生成的子作用域继承了这个函数可以直接使用。

html

    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Title</title>
        <script src="js/angular.js"></script>
    </head>
    
    <body ng-app="a">
    <div ng-controller="WorldCtrl">
        <h1>Population, {{population}}!</h1>
        <li ng-repeat="counry in countries">
            {{counry.name}} has {{counry.population}},
            {{worldsPercentage(counry.population)}} % of the World's population
        </li>
    </div>
    </body>
    
    
    <script language="JavaScript" type="text/javascript">
        var moA = angular.module("a", []);
    
        moA.controller("WorldCtrl", function ($scope) {
            $scope.population = 7000;
            $scope.countries = [{name: 'china', population: '70'}, {name: 'china1', population: '701'}]
            $scope.worldsPercentage = function (countryPopution) {
                return (countryPopution / $scope.population) * 100
            }
        });
    </script>
    
    </html>

## 作用域继承的风险，$parent与对象属性
先给出以下三个例子,父作用域是rootScope，子作用域分别是HelloCtrl、HelloCtrlOne、HelloCtrlTwo

html

    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Title</title>
        <script src="js/angular.js"></script>
    </head>
    
    <body ng-app="a" ng-init="name = 'World';thing = {name:'World'}">
    <h1>Hello,{{name}},{{thing.name}}</h1>
    <div ng-controller="HelloCtrl">
        <label>Say hello to :
            <input type="text" ng-model="name">
        </label>
        <h1>Hello, {{name}}!</h1>
    </div>
    
    <div ng-controller="HelloCtrlOne">
        <label>Say hello to :
            <input type="text" ng-model="$parent.name">
        </label>
        <h1>Hello, {{name}}!</h1>
    </div>
    
    <div ng-controller="HelloCtrlTwo">
        <label>Say hello to :
            <input type="text" ng-model="thing.name">
        </label>
        <h1>Hello, {{thing.name}}!</h1>
    </div>
    </body>
    
    
    
    
    <script language="JavaScript" type="text/javascript">
        var moA = angular.module("a", []);
        moA.controller("HelloCtrl", function ($scope) {
        });
    
        moA.controller("HelloCtrlOne", function ($scope) {
        });
    
        moA.controller("HelloCtrlTwo", function ($scope) {
        });
    </script>
    </html>
    
##作用域层级与事件系统
留个坑以后填

##作用域的生命周期
1、作用域在不需要后会被销毁，在其中暴露的模型和函数都会被回收

2、作用域通常会依赖作用域创建指令而创建和销毁，也可以调用$scope.$new() 或者 $scope.$destroy()方法，手动创建或销毁
