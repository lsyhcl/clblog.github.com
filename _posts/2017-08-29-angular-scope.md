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
1、某些指令会创建子作用域。例如ng-repeat。下面例子中的&lt;li&gt;标签下的population，就在新生成的作用域(scope)中

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
            {{counry.name}} has {{counry.population}}
        </li>
    </div>
    </body>
    
    
    <script language="JavaScript" type="text/javascript">
        var moA = angular.module("a", []);
    
        moA.controller("WorldCtrl", function ($scope) {
            $scope.population = 7000;
            $scope.countries = [{name: 'china', population: '70'}, {name: 'china1', population: '701'}]
        });
    
    
    </script>
    </html>

