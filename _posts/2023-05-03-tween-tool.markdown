---
layout: post
title: Tween Tool
date: 2023-05-03 22:41:20 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: TweenTool/UnityLogo.png
---

# Introduction

A tween tool is a plugin that allows us to perform interpolations in a more concise way if compared to using Unity animation system. They are very useful for situations where creating an Animator Controller and an Animation feels too much of an overkill and adds more overhead than functionality. I came across some different approaches to implementing them, and decided to create my own version, based on my idea of the role of this tool while developing games and it's relation with what Unity already offers us.

---

## Requisites

I decided the list of requisites i wanted for my tween tool:

* A tween/interpolation must be playable and stoppable by any code that has access to a reference to a Tween object.

* A Tween object is **not** an Monobehaviour.

> In my opinion, tweens are bound to their owners when it comes to the MonoBehaviour lifecycle and because of that i don't believe they should be behaviours themselves, instead, they are composed inside a MB that wants their functionality. This also respects my ideal that they must be simple.

* The Tween object will update together with an Monobehaviour object that is set as its owner, running this update in a Unity Coroutine.

> Long story short, a Monobehaviour declares a tween object, this tween object receives an owner (usually this owner is the declaring MB itself, but thats not a rule!) and when this tween is set to run, it runs inside a Coroutine from the "owner", hence the name.

* The Tween object runs for a specified duration, and allows for a delayed start.

* The Tween object handles subscribing and unsubscribing to the following events:

1.  *The start of the tween: triggers immediately as soon as the tween run method is called.*

2.  *The end of the tween delay: triggers after a delay set, after the tween has started.*

3.  *The end of the tween: triggers when the tween ends. Also, the tween can be set to trigger this event when being forcefully stopped.*

> Usually tweens run for more than one frame, so those events are a must have for easier code control.  

* The Tween object can be set to update ignoring or not the game time scale.

> Its quite common to implement pause in a game by setting the time scale to 0, though this approach has some problems. In order to allow tweens to work in this scenario, it is possible to set them to use unscaledDeltaTIme instead of deltaTime.

* The Tween object must inform if it is currently running.

* The interpolation must be configurable by using custom curves.

> Similar to animations, sometimes we may not want a simple linear interpolation between values to be performed. Instead of interpolating two values using a simple factor X that varies from 0 to 1 linearly, it is usefull for us that this value X is instead used to evaluate a custom curve f(X) = Y, and Y is used instead. For example, instead of interpolating a position from A to B linearly, we can define a curve f(X) = X² that will make this position to vary as if it is accelerating as X gets close to 1.

* The Tween object must support finite and infinite looping, as well as handling subscribing and unsubscribing to events triggered when each loop ends.

> Looping is something very common when animating, so it is important that tweens are able to loop, and that allows for the creation of a new event to keep track of the tween progress, an event that informs when each loop ends.

* The Tween object must support grouping, allowing multiple tweens to run in sync, **as long as they have the same owner**.

> Though the goal is to keep tweens simple, being able to group some simple tweens together and let them play at the same time ( of with a delayed start for each one ) is a great feature to sync them and prevent code bloating. A reasonable rule here is that all tweens must be owned by the same Monobehavior so they can run together. If each tween had different owners, some of those could be inactive when the tween is asked to run, causing weird results.

* It must be easy to create new and custom tweens for any exposable property, that can be interpolated.

> Last but not least, this is the most important point. It MUST be easy to create a new custom tween over an exposable and interpolable property. If this is not the case, the whole goal of this tool is meaningless.

---

## Results
Before presenting details of the implementation and the reasoning behing each requisite, i will show some gifs of the final results applied to an example scene. Do note that the quality of the gifs are not ideal because they are .mp4 videos converted to .gif, but this lack of quality does not prevent what is relevant to be understood.

**Position Tween**
![Position Tween]({{site.baseurl}}/assets/img/TweenTool/Position.gif)

**Rotation Tween**
![Rotation Tween]({{site.baseurl}}/assets/img/TweenTool/Rotation.gif)

**Scale Tween**
![Scale Tween]({{site.baseurl}}/assets/img/TweenTool/Scale.gif)

**Loop Tween**
![Loop Tween]({{site.baseurl}}/assets/img/TweenTool/Loop.gif)

**Group Tween**
![Group Tween]({{site.baseurl}}/assets/img/TweenTool/Group.gif)

---

## Implementation
As shown in the following UML, the four main definitions used while implementing the tween are:
 -  **ITween**:  interface that defines most generic base methods
 - **Tween<T,U>**: abstract class that implements ITween and adds custom interpolation curves and a very important abstract method, Evaluate, that is the one responsible for calculating the current value U for the property of the same type exposed by T. This method calculates the value for U considering the current tween progress that ranges from 0 to 1 (before being modified by the custom curve) and the initial and target U values set when constructing the class (or initialized via serialized unity properties). Do note that T must have an exposable and interpolable field U.
 - **LoopTween<T,U>**: abstract class that inherits from Tween<T,U> with an overriden RunRoutine, methods to subscribe to new events related to the end of each loop, and a field that defines loop amount.
 - **TweenGroup**: concrete class that implements ITween. Exposes methods to handle a list of tweens for a specific owner and a start delay for each one of them. The overriden RunRoutine of this class works by playing each individual tween after the specified startGroupDelay that each one is associated to. 
 
![Tween UML]({{site.baseurl}}/assets/img/TweenTool/Tween.png)

 In order to create a new tween concrete class, this class is required to define who are T and U, where U is an exposed and interpolable value that T owns, and define the interpolation by implementing the method Evaluate. Do note that this concrete class will either implement Tween<T,U> or LoopTween<T,U>. It follows the example implementation of the PositionTween, where T is a Transform and U a Vector3.

{% highlight ruby %}
public class PositionTween : Tween<Transform, Vector3>
{
	public PositionTween(MonoBehaviour monoBehaviour,
	Transform entity,
	Vector3 initialValue,
	Vector3 targetValue,
	float duration)
	: base(monoBehaviour, entity, initialValue, targetValue, duration) { }

	protected override void Evaluate(float? progressOverride = null)
	{
		float t;
		t = progressOverride != null? progressOverride.Value : progressWithCurveApplied;
		entity.position = (1f - t) * initialValue + (t) * targetValue;
	}
}
{% endhighlight %}