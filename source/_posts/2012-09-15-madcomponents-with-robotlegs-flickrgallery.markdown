---
layout: post
title: "MadComponents with RobotLegs: FlickrGallery"
date: 2012-09-15 20:58:00 +0200
comments: true
categories: programming
tags: [air, madcomponents, robotlegs]
---

->![MadRobot](/images/madrobot.png =157x252)<-

## Introduction

During last days I was playing with amazing AS3 library [MadComponents](http://code.google.com/p/mad-components/) - lightweight open-source UI framework for Flash. This library is maintained by [Daniel Freeman](http://madskool.wordpress.com/about-2/) and he claims that "MadComponents" targets mobile apps via Adobe AIR. And I could say that the library proves that point: it's fast and provides ready-to-use components, which fit to mobile UI, out of the box. One of the great features is a declarative UI markup: just feed framework with correct XML and it produces components' tree that automatically layouts depending on device screen dimensions.

{% codeblock Example of XML layout lang:as3 %}
protected const LAYOUT: XML =
	<horizontal alignH="right">
		<input id="searchTermInput" width="150" />
		<button id="submitSearchButton">search</button>
	</horizontal>;
{% endcodeblock %}

I've created prototype for my commercial project very quickly utilizing MadComponents. Since prototype phase had satisfied my needs I've decided to introduce the library with my favourite ActionScript MVC framework [RobotLegs](http://www.robotlegs.org/).

## Modifying the architecture of existing Flex Robotlegs-based FlickrGallery

In this article I will share with you my experience of replacing Flex controls with corresponding MadCompontets in Joel Hooks' [robotlegs-examples-FlickrGalleryNoMeta](https://github.com/joelhooks/robotlegs-examples-FlickrGalleryNoMeta). You can read more about "nometa" approach of using RobotLegs in the Joel's ["Do we really need THAT much metadata in AS3? Not with Robotlegs…"](http://joelhooks.com/2010/06/16/do-you-need-metadata-as3-robotlegs/). 

*Note. This example app is designed for desktop, not mobile, hence that it's just general purpose article, not mobile tutorial for MadComponents and Robotlegs.*

First of all let's describe structure of original FlickrGalleyNoMeta to decide what exactly should be reimplemented with MadComponents.

->![Package diagrame before](/images/FlickrGallery-package-diagram.png)<-

In the scheme [1] there are shown package diagram of the app. At the top of root package we have two classes: ImageGalleryContext and RobotLegsFlickrGallery (context view - in the terms of RobotLegs). With green background I've marked classes that are Flex-based: context view and UI components. But as we see, "mediators" depends on Flex components. So first of all, let's create abstrct interfaces which expose contract for interaction with UI: new package "api" inside "components". Then move Flex based components to "flex" subpackage and create "mad" package for MadComponents UI. Also I need to have new Mad-based main view and context (which maps mediators to Mad-components).

->![Package diagrame after](/images/FlickrGallery-package-diagram-after-refactor.png)<-

I've extracted base context class `BaseGalleryContext` and derived two new ones for Flex and Mad. Now our application structure is agnostic to concrete UI library. If we want to compile Flex version, then just pick `MadRobotlegsFlickrGallery` as swf main class, or for MadComponents version - `MadRobotlegsFlickrGallery`. And this is real power of MVC - easy replacing of layers implementation: so mediators depends only on UI interfaces.

Here how I map mediators to "mad" components:

{% codeblock BaseGalleryContext.as https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/src/tip/src/org/robotlegs/demos/imagegallery/BaseGalleryContext.as %}
import org.robotlegs.demos.imagegallery.views.components.mad.*;
import org.robotlegs.demos.imagegallery.views.components.api.*;

// ... more code ...

override public function startup():void
{
	//map view
	mediatorMap.mapView(GalleryView, GalleryViewMediator, IGalleryView);
	mediatorMap.mapView(GallerySearch, GallerySearchMediator, IGallerySearch);
	mediatorMap.mapView(GalleryLabel, GalleryLabelMediator, IGalleryLabel);
	
	super.startup();
}

// ... more code ...
{% endcodeblock %}

The third parameter of `mapView` specifies the type that mediator depends on. For example, `GalleryViewMediator` depends on `IGalleryView` type. The injector will automatically pass the view into constructor.

{% codeblock GalleryViewMediator.as https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/src/tip/src/org/robotlegs/demos/imagegallery/views/mediators/GalleryViewMediator.as %}
public class GalleryViewMediator extends Mediator
{
	private var view:IGalleryView;

	public function GalleryViewMediator(galleryView:IGalleryView)
	{
		view = galleryView;
	}

	// ... more code ...
}
{% endcodeblock %}

## Implementing UI with MadComponents

Ok, application's backbone is constructed correctly, but it misses the main part - actual Mad UI! So lets implement root view and view interfaces: `IGalleryLabel`, `IGallerySearch`, `IGalleryView`.

General layout consists of header, search prompt and gallery view:

{% codeblock MadRobotlegsFlickrGallery.as https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/src/tip/src/MadRobotlegsFlickrGallery.as %}
protected const LAYOUT:XML =
	<vertical xmlns:mad="org.robotlegs.demos.imagegallery.views.mad"
		gapV="0" gapH="0" paddingV="0" border="false">
		<mad:GalleryHeader />
		<mad:GallerySearch id="search" />
		<mad:GalleryView id="gallery" />
	</vertical>;
{% endcodeblock %}

To split layout into seperate files, I've utilized the [UI.extend() technique](http://madskool.wordpress.com/2012/01/12/extending-madcomponents/), where custom container componet derives from `UIPanel` (class from ExtendedMadnessLib). In order to recognize new components by framework, we should put "XML tag name <=> Class" mapping before whole UI creation:

{% codeblock MadRobotlegsFlickrGallery.as https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/src/tip/src/MadRobotlegsFlickrGallery.as %}
private function onAddedToStage(e:Event):void
{
	UI.extend(
		["GalleryHeader", "GallerySearch", "GalleryView", "GalleryLabel"],
		[GalleryHeader, GallerySearch, GalleryView, GalleryLabel]);
		
	UI.PADDING = 0;
	UIe.create(this, LAYOUT);
}
{% endcodeblock %}

Let's look closer to the simplest component `GalleryHeader`. There ar no surprising things in it, except the pattern, which I use for every custom container component. The class' constructor takes same range of parametres as parent class and passes all given arguments into parent's constructor, except `xml` parameter. Instead of external XML file, I pass internal `LAYOUT` constant.

{% codeblock GalleryHeader.as https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/src/tip/src/org/robotlegs/demos/imagegallery/views/components/mad/GalleryHeader.as %}
package org.robotlegs.demos.imagegallery.views.components.mad 
{
	import com.danielfreeman.extendedMadness.UIPanel;
	import com.danielfreeman.madcomponents.Attributes;
	import flash.display.Sprite;
	import flash.utils.getQualifiedClassName;
	
	/**
	 * @author Bohdan Makohin
	 */
	public class GalleryHeader extends UIPanel
	{
		protected const LAYOUT: XML =
			<horizontal>
				<image>{getQualifiedClassName(Assets.RobotLegsLogo)}</image>
				<label id="headerLabel"><font size="24">
					<b>Robotlegs: Image Gallery Demo</b>
				</font></label>
			</horizontal>;
		
		public function GalleryHeader(screen:Sprite, xml:XML, attributes:Attributes=null, row:Boolean=false, inGroup:Boolean=false) 
		{
			super(screen, LAYOUT, attributes, row, inGroup);
		}
	}

}
{% endcodeblock %}

Described approach allows to reuse such components and encapsulate implementation details, in contrast to [Michael Martinez' practice](http://caffeineindustries.com/2011/09/as3-madcomponents-tutorial-series-separation-of-concerns/), where separate class hosts publicly accessed static variable with XML. Let's define pros and cons of both methods (for ease of explanation I will call Martinez' technique "static" and mine - "extension").

### Pros and cons of "static" and "extension" techniques to extending MadComponents

MadComponents framework has strong documentation on how to compose single layout for whole application, where XML tree is constructed during compile-time. And "static" technique utilized same scheme:

{% codeblock lang:as3 Example of "static" layout %}
protected static const NAVIGATION:XML = 
	<navigation id="nav" colour="#C100000" title="Home">
		{LIST}
		{Page0.LAYOUT}
		{Page1.LAYOUT}
	</navigation>;
{% endcodeblock %}

Here `Page0` and `Page1` are custom component classes. This doesn't broke the mentioned principle of static XML markup.

On the other hand with "extension" technique we don't have single XML markup for whole application, but many separate XML trees for each custom composite component. This approach hides the internals of our component, and therefore reduces tight coherence, which is important characteristic for most software.

With "static" technique the parent control is responsible for initialization of custom control:

{% codeblock lang:as3 Example of "static" initialization %}
// Create the main UI or "Landing Page"
UI.create(this, NAVIGATION);

// Initialize "views"
Page0.initialize();
Page1.initialize();
{% endcodeblock %}

So whenever we want to use such control, it is necessary to reference it at least twice: in XML markup and initialization code. Such approach could enforce mistakes, when we want to replace `Page1` with, for example, `Page3` - remember to change code in two places.

Strictly speaking, "static" approach is just procedural wrapper over XML - we don't actually do OOP, but have bunch of static methods within class declaration. We can't benefit from instantiating such controls, because everything is located in static context. Hence that is not easy to integrate with MVC - we should manually instantiate and dispose mediators (Robotlegs), or create dummy Sprite object to be adapter between mediators and custom Mad components.

The "extension" method gives us possibility to use `protected` methods and fields of UIPanel (or UIForm); we could override some methods for our needs. But one of the main benefits is, that we do OOP - so it's easy to integrate with MVC, create as many instances of component as we need. Our custom control is self-contained building block (the question is: do you need it to be such?).

### Typed model and UIList's custom renderer

Bad news : by default you cannot bind properties of user-defined class within `UIList` custom renderer. `UIList` recognizes only dynamically added properties, not class-defined. So if you have defined some properties in your model class and list renderer references them - it will no be bound. The reason of such behavior is that implementation of of UIList uses "[for..in](http://help.adobe.com/en_US/ActionScript/3.0_ProgrammingAS3/WS5b3ccc516d4fbf351e63e3d118a9b90204-7fcf.html#WS5b3ccc516d4fbf351e63e3d118a9b90204-7f5b) to iterate through model.

{% codeblock GalleryImage.as https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/src/tip/src/org/robotlegs/demos/imagegallery/models/vo/GalleryImage.as %}
public class GalleryImage
{
	protected var _URL:String;
	protected var _thumbURL:String;
	protected var _selected:Boolean;

	public static const EMPTY: GalleryImage = new GalleryImage();
	
	public function get selected():Boolean
	{
		return _selected;
	}

	public function set selected(v:Boolean):void
	{
		_selected = v;
	}

	public function get thumbURL():String
	{
		return _thumbURL;
	}

	public function set thumbURL(v:String):void
	{
		_thumbURL = v;
	}

	public function get URL():String
	{
		return _URL;
	}

	public function set URL(v:String):void
	{
		_URL = v;
	}

}
{% endcodeblock %}

I have two solutions for this issue:

1. Create new list class derived from `UIList` (or `UIListHorizontal`) and override it's `fillInValues` method to directly cast model to appropriate class and set the values of properties. Such custom list could take a delegate function, which knows details of model type.
2. Create new object with dynamic properties, which corresponds to typed model:

{% codeblock GalleryView.as https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/src/tip/src/org/robotlegs/demos/imagegallery/views/components/mad/GalleryView.as %}
protected static function imageListToArray(list: IList): Array
{
	var images: Array = new Array();
	
	for (var i:int = 0; i < list.length; ++i )
	{
		var image: GalleryImage = GalleryImage(list[i]);
		images.push(
		{
			URL: image.URL,
			thumbURL: image.thumbURL,
			selected: image.selected
		});
	}
	
	return images;
}

// ... some code ...

public function set dataProvider(value:IList):void 
{
	// UIList doesn't recognize user-defined properties
	_dgThumbnails.data = imageListToArray(value);
}
{% endcodeblock %}

First solution forces you to reimplement internals of MadComponents, which is not great in the case internals will be modified later. Second solution depends only on public interface of `UIList`, but requires twice (roughly) as much memory for model objects. So it's your decision, which solution to choose.

### You cannot access children views in Mediator.onRegister()

This problem appears when you want access view's children within `onRegister()` function of view's mediator. But why? It happens because the mediator is created right after view's `ADDED_TO_STAGE` event is fired. However MadComponents contaiers are added to it's parent display list in the first line of it's constructor, and after that children creation takes place - so mediator is created between those two phases. I've met this problem in `GalleryLabelMediator`.

{% codeblock GalleryLabelMediator.as https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/src/tip/src/org/robotlegs/demos/imagegallery/views/mediators/GalleryLabelMediator.as %}
override public function onRegister():void
{
	addContextListener(GallerySearchEvent.SEARCH, handleSearch );

	view.text = "interestingness";
	view.setVisibility(service.searchAvailable);
}
{% endcodeblock %}

Here mediator tries to access view's label text, but it doesn't exist at the time of `onRegister()` call.
In order to handle this, you should consider to perform initialization step of view inside view's constructor using data, passed from mediator.

{% codeblock GalleryLabel.as https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/src/tip/src/org/robotlegs/demos/imagegallery/views/components/mad/GalleryLabel.as %}
package org.robotlegs.demos.imagegallery.views.components.mad 
{
	import com.danielfreeman.extendedMadness.UIPanel;
	import com.danielfreeman.madcomponents.Attributes;
	import com.danielfreeman.madcomponents.UI;
	import com.danielfreeman.madcomponents.UILabel;
	import flash.display.Sprite;
	import flash.events.Event;
	import flash.text.TextFormat;
	import flash.text.TextFormatAlign;
	import org.robotlegs.demos.imagegallery.views.components.api.IGalleryLabel;
	
	/**
	 * @author Bohdan Makohin
	 */
	public class GalleryLabel extends UIPanel implements IGalleryLabel 
	{
		private var _text: String;
		private var _label: UILabel;
		
		protected const LAYOUT: XML =
			<vertical>
				<label id="label" alignH="right">
					<font color="#ffffff" size="12"><b></b></font>
				</label>
			</vertical>;
		
		public function GalleryLabel(screen:Sprite, xml:XML, attributes:Attributes=null, row:Boolean=false, inGroup:Boolean=false) 
		{
			super(screen, LAYOUT, attributes, row, inGroup);
			
			_label = UILabel(findViewById("label"));
			_label.text = _text;
		}
		
		public function get text():String 
		{
			return _text;
		}
		
		public function set text(value:String):void 
		{
			_text = value;
			if (_label)
			{
				_label.text = _text;
				doLayout();
			}
		}
		
		public function setVisibility(visibility :Boolean):void 
		{
			this.visible = this._inGroup = visibility ;
		}
	}
}
{% endcodeblock %}

## Instead of conclusions

### Compile and try
"Madcomponents-robotlegs-FlickrGalleryNoMeta" project is accessible at [BitBucket mercurial repository](https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/overview). You could clone the repo or download [archive with sources](https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/get/tip.zip). If you want to compile project by yourself, you may use FlashDevelop's project file. For those, who want just to play with application, I've uploaded the [madcomponentsrobotlegsFlickrGalleryNoMeta.swf](https://bitbucket.org/ajukraine/madcomponents-robotlegs-flickrgallerynometa/downloads/madcomponentsrobotlegsFlickrGalleryNoMeta.swf) file, built with AIR 3.4. It could be opened in latest flash player (in order to activate layout, resize application window a bit after opening  the player).

### I was lazy
I was lazy to achieve same look-and-feel as in original app. Therefore I've omitted to implement nice tweening effect for image switching - just utilized standard `UIImageLoader` component. Besides that I had hard time with skinning thumbnails list (`UIListHorizontal`) at the bottom, so it doesn't have background and proper selection illumination. 

### Thoughts
It was my first try to integrate RobotLegs with MadComponents. And I liked it. Despite MadComponents has some minor drawbacks (lack of data binding, inflexible component lifetime) it stands alone in the area of fast, lightweight and comprehensive UI frameworks for mobile via Adobe AIR. As far as I know, Daniel Freeman is going to release new major version of the library, which will take advantage of Stage3D and direct rendering mode.

### Summing up
If you was doubt about using RobotLegs with MadComponents, I'd rather you to hurry up and evaluate this post in terms of your requirements. But if you prefer stay away, then you should know that I'm going to prepare new article with more in-depth dive into mobile development with MadComponents and RobotLegs.

->![Scrrenshot of app](/images/mad-flickrgallery-screenshot.jpg)<-