# **Table of contents**
* [Goal](#goal)
* [Basic Structure](#basic_structure)
* [When to use this pattern](#when_to_use_this_pattern)
* [How NOT to use this pattern](#how_not_to_use_this_pattern)
    * [How to (Partially) avoid these bad things from happening in the first place.](#avoid_these_bad_things)
        * [Composition over inheritance](#composition_over_inheritance)
        * [Pitfalls with Composition over inheritance](#pitfalls_composition_over_inheritance)
* [When NOT to use the "Template design pattern"](#when_not_to_use_template_design_pattern)
* [So what to actually do](#so_what_to_actually_do)

<br></br>
<a name="goal"/>
# Goal
Reduce code duplication and have a clear structure for the logic.

This is suited for cases where several components have some similar "steps" in them that must always happen, and should happen in a specific order.

The "workflow" is as follows:
1. Given the multiple components at hand, identify the common logic.
2. Extract this common logic into a single function (Called the "skeleton").
3. This common logic will leave "hooks" ("holes") which will be filled by each different component.


Notice that the common "steps" (And their matching order) should be an **invariant** of all the components at hand.
> NOTE: The name "template" means literally that the pattern looks like a template. It has nothing to do with C++'s templates (`template<>`) :)

<br></br>
<a name="basic_structure"/>
# Basic structure
The basic structure is as follows:
1. An abstract base class with the "skeleton" of the logic in it. The "skeleton" in this abstract class will call into virtual functions.  
This abstract class _may_ have some default implementation in cases where this step is an does not change between the different concrete logics.
    ``` cpp
    class BaseLogic {
    public:
        // The actual "template"
        void action() {
            subaction_1();
            subaction_2();
            subaction_3();
        }
    
        virtual ~BaseLogic() = default;
    
    private:
        // Sometimes changes between components, so allow overriding it
        virtual void subaction_1() {
            // Default subaction 1
        }
    
        // No default subaction_2
        virtual void subaction_2() = 0;
        
        // subaction 3 should never change
        void subaction_3() {
            // subaction 3
        }
    };
    ```
    
    Here, `do_action()` is the "template" of the logic: It enforces that first `subaction_1()` is called, then `subaction_2()` and then `subaction_3()`.  
    
2. Each concrete logic will inherit from this class, and override the virtual functions ("Fill in the holes") with their own logic.
    ``` cpp
    class ConcreteComponent1 : public BaseLogic {
    private:
        virtual void subaction_1() override {
            // Concrete subaction 1
        }
    
        virtual void subaction_2() override {
            // Concrete subaction 2
        }
    };
    ```

<br></br>
<a name="when_to_use_this_pattern"/>
# When to use this pattern
As one can see, this pattern is a pretty powerful way to avoid code reuse.

Also, since the code for the "template" class will not be changed, it fits well into the "open/closed principle". Good times.

Therefore, using this pattern when a few components share the same logic can result in better readability and maintainability of the code.  

<br></br>
<a name="how_not_to_use_this_pattern"/>
# How **NOT** to use this pattern
If you stop here and implement all your code around this pattern like it showed up until here, you'll more often than not get yourself into a mess, and realize it only when its too late:


Usually, these things can go bad when implementing this pattern with no care:  
1. At some point in the future you'll probably encounter a need to add a component to the inheritance tree, but it doesn't _quite fit_ into the skeleton in the base class.  
    What will usually happen in this case is either (Depending on how well the new logic plays with the existing code):
    1. If the new logic defied something that was previously thought as being an invariant, this code now will be moved from the template class into each concrete logic.
    2. If the new logic requires a new "step" that wasn't thought as being needed before, new "hooks" will need to be added to the template class, and all the existing concrete components will have to implement this new "step". 

    Doing either one is not going to be a good thing, as this will force you to go over the existing components and make sure you will not break any of them.  
    
2. During the implementation you'll probably notice that some of the concrete components have some similar behaviour.
In this case, the most easy thing to do is to move this common behaviour into a "checkpoint" in the inheritance tree, and have the concrete components (That share that specific behaviour) inherit from this "checkpoint".  
    To illustrate (Excuse me for the lack of better names):
    ``` cpp
    class BaseLogic {
    public:
        void action() {
            subaction_1();
            subaction_2();
            subaction_3();
        }
    
        virtual ~BaseLogic() = default;
    
    private:
        virtual void subaction_1() = 0;
        virtual void subaction_2() = 0;
        virtual void subaction_3() = 0;
    };
    
    // All components that share a similar structure to 'subaction_2()'
    class CheckpointComponent : public BaseLogic {
    private:
        virtual void subaction_2() override { ... }
    };
    
    class ConcreteComponent1 : public CheckpointComponent {
    private:
        virtual void subaction_1() override { ... }
        virtual void subaction_3() override { ... }
    };
    ```
    
    The inheritance tree will look something like:  
                                                                     **BaseLogic**  
                                                                    /                 \\  
                                                                   /                   \\  
                                **CheckpointComponent**          ConcreteComponent3  
                                              /                       \\  
                                             /                                              \\  
                        **ConcreteComponent1**        ConcreteComponent2  

    
    This can quickly result in unreadable and unmaintainable code: In order to see what `ConcreteComponent1` does, one has to traverse to **whole** inheritance tree in order to see if some classes along the way give their own implementation to some subaction, which is really not something you want to do.  
    While this might seem trivial when you read it here, be wary that this behaviour can manifest itself **later in development** - Only after a few weeks/months/years, when more concrete components will be implemented, where such similarities will pop up (And the need to have these "checkpoints")

<br></br>
<a name="avoid_these_bad_things"/>
## How to (Partially) avoid these bad things from happening in the first place
We can partially avoid the 2nd thing from happening by thinking on how this pattern is implemented, which is - **Code reuse via inheritance**.

<br></br>
<a name="composition_over_inheritance"/>
### Composition over inheritance
Inheritance forces a rigid structure for the implementation, and the only way to re-use code is via inheriting from the class that implements this behaviour.

With composition, the implementation of the different subactions will be delegated to **data members** containing the logic to implement each subaction:
``` cpp
class ISubaction1 {
public:
    virtual ~ISubaction1() = default;

    virtual void do_action() = 0;
};

class ISubaction2 {
public:
    virtual ~ISubaction2() = default;

    virtual void do_action() = 0;
};

class ISubaction3 {
public:
    virtual ~ISubaction3() = default;
 
    virtual void do_action() = 0;
};

class ComponentLogic {
public:
    ComponentLogic(std::unique_ptr<ISubaction1> action_1,
                   std::unique_ptr<ISubaction2> action_2,
                   std::unique_ptr<ISubaction3> action_3)
     :  m_action_1(std::move(action_1)),
        m_action_2(std::move(action_2)),
        m_action_3(std::move(action_3))
    { }

    void action() {
        m_action_1->do_action();
        m_action_2->do_action();
        m_action_3->do_action();
    }

private:
    std::unique_ptr<ISubaction1> m_action_1;
    std::unique_ptr<ISubaction2> m_action_2;
    std::unique_ptr<ISubaction3> m_action_3;
};

class ConcreteAction1 : public ISubaction1 {
public:
    virtual void do_action() override { ... }
};

class SameConcreteAction2 : public ISubaction2 {
public:
    virtual void do_action() override { ... }
};

class ConcreteAction3 : public ISubaction3 {
public:
    virtual void do_action() override { ... }
};

ComponentLogic concrete_component1 { std::allocate_unique<ConcreteAction1>(),
                                     std::allocate_unique<SameConcreteAction2>(),
                                     std::allocate_unique<ConcreteAction3>(),
                                   };

ComponentLogic concrete_component2 { std::allocate_unique<AnotherConcreteAction1>(),
                                     std::allocate_unique<SameConcreteAction2>(),
                                     std::allocate_unique<AnotherConcreteAction3>(),
                                   };
```

And thus, the code reuse was fixed by simply given the **same** `ISubaction2` class to the different components.  
Also, in order to check what subactions `concrete_component2` does, one simply needs to check how it's initialized (Which can be moved to a factory).  

Nice, clean and highly maintainable :)

<br></br>
<a name="pitfalls_composition_over_inheritance"/>
### Pitfalls with Composition over inheritance
I'd argue that sometimes, using composition over inheritance can result actually in **less** readable code - If there are a lot of subactions for each logic, then in order to grasp what each concrete component does can be quite tedious, as it will require "jumping around" multiple classes and different logics due to the relevant code being "fragmented" around the code base.

Further more, let's say that now there are new required components, all of which share a __somewhat__ similar `subaction2()`.  
How similar are the different `subaction2()`? Enough that one might be tempted to apply the "Template design pattern" into `subaction2()` as well, again using composition.

Repeat it a few times and you'll quickly end up with dozens of such "utility classes" and an over engineered mess that only you will probably understand, as wrapping your head around how everything works in the **big picture** when so much classes are interpolated can be challenging.

If instead in the first place, each component had all of its logic inside it, then going through the code can become easier.
<br></br>
Again I'm not saying that one way is **always** better than the other.  
The point is to show you the possible ways to achieve the goal - At the end you're the one that should think about the trade-offs between the different design choices, and come up with the best one for your **specific** need.

<br></br>
<a name="when_not_to_use_template_design_pattern"/>
# When NOT to use the "Template design pattern"
Sometimes, I'd argue that one shouldn't even use this pattern in the first place:

As stated, the main reason to use this pattern is to avoid code reuse.

Now, it's important to know that **_some code duplication is fine!_**

It's far better to have code duplication, if the design pattern that will be used to "fix" it does not correctly capture reality - Since while you might see the benefit of this pattern in the current code and existing requirements, it's important to see how the code will change if a new requirement will be added, one that will break the "invariants" that are currently in place (For this pattern, the invariants are the order of functions)
<br></br>
Otherwise, if new requirements that don't quite fit into the existing pattern will be added, each new change will "bend" the abstraction (The template design pattern, in this case) just a bit, and overtime the whole code will become an unmaintainable mess.

<br></br>
<a name="so_what_to_actually_do"/>
# So what to actually do
Don't get me wrong, I'm not advocating against implementing this pattern using inheritance.

It can have great benefits to the overall structure of the code, but only if it was carefully thought about, and implemented properly.

Therefore, I'd rewrite the "workflow" mentioned in the first paragraph as follows:
1. First, think about whether or not you even need to use this pattern. Maybe the code duplication that it will save is so small and minor, that all the hassle around it will not be worth it.
2. If you indeed decide on using this pattern, make **absolutely sure** you got all the edge cases before you actually implement the template class (This is usually impossible to get perfect as new demands can come up a few years from now - Ones you didn't even think were possible).
3. If you decide on implementing this with inheritance, make the inheritance tree **shallow** - Make sure not to have components that inherit from other components (Avoid having these "checkpoint components"). Instead, make sure that each component only inherits directly from the base logic.
4. Think if some components are an entity of their own, which can be used in contexts other than where the components will be used. If so, think about implementing this pattern using composition instead.
5. If indeed you decide to implement this pattern using composition, make sure the number of different objects (And thus, interfaces) is small - No one likes to see an object that accepts 20 interfaces in order to work.
