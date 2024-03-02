---
title: "Hellbound: Reflection data to Editor window"
date: 2024-3-2 18:06:00 +0800
#image: "/assets/img/HellboundEditor/EditorOverview.mp4"
---

# Introduction
During the development of Hellbound, we used [Entt](https://github.com/skypjack/entt), It is a amazing ECS library that has made our lives easier.
But when it comes to generic things like an editor window for all components or serialization, how would we deal with that?

I came up with a solution based around Entt's Reflection library.

But before I wil show you how it's made, I will show you my library in action.

The video below showcases some of the options you have with Property drawing.
- Many types (general data types, `glm::vec3`, `entt::entity`)
- Option to mark elements `read-only`
- Draw vectors as `color pickers` or `normalize` them
<video controls>
  <source src="/assets/img/HellboundEditor/EditorOverview.mp4" type="video/mp4">
Your browser does not support the video tag.
</video>

The next video showcases some of the other functionality,
More generic functionality which was requested by the team!
- Search filter for component types
- Removing components
- Resizing entities list (Was highly requested during hellbound development.) 

<video controls>
  <source src="/assets/img/HellboundEditor/SearchFilterAndRemove.mp4" type="video/mp4">
Your browser does not support the video tag.
</video>

# Options
Before I talk about my solution in depth, I want to share what other options I considered.
Because without a good reason, I would not have created my solution.

## Design principles
Some of my requirements for an editor solution:

- Easy to expose variables to the editor
- Idealy able to expose variables without writing Imgui code.
- Easy markup for variables (Readonly, DrawingStyle, etc.)
- All within `<Component>.h` and `<component>.cpp`
- Don't make it dependent on many libraries
- Easy to Expose new variable types


## Green-Sky Editor
To my knowledge, there is only one library out there that connects Dear Imgui with ENTT.
[You can see their example usage here](https://github.com/Green-Sky/imgui_entt_entity_editor?tab=readme-ov-file#example-usage)
At first glance, it looks like it does exactly what I want from it.

There are a few things that go against my design principles with this library:

The user has to write ALL Imgui code themself, This is not something a gameplay programmer should have to spend time on.
- Manually registering the type in another file
    - We can fix this with a macro.
- Imgui code written in header files. Not ideal for compile times. (We can work around this, but it adds more steps for people to get UI for a component working!)
- We can't extend this for things other than UI.


## Entt Reflect
Entt comes with its own [runtime reflection system]( https://github.com/skypjack/entt/wiki/Crash-Course:-runtime-reflection-system )
If this works well, It would be ideal to use because it's made with entt in mind and reduces the number of libraries we include in the engine.

Creating types with this is relatively simple, anywhere you can do the following to define a type:
```cpp
auto factory = entt::meta<my_type>().type("reflected_type"_hs);
//Register variables
factory.data<&my_type::static_variable>("static"_hs);
factory.data<&my_type::data_member>("member"_hs);
factory.data<&global_variable>("global"_hs);
//Register function
factory.func<&my_type::member_function>("member"_hs);
```
There is one issue that the library doesn't cover. When should we call this?
The examples show it all done in the main function, which breaks one of the `Design principles` I have for the library.

But as it's just code which we can call anywhere, we can solve this ourselves!

To loop over all types of variables we can do something like this (Again from anywhere in the code base)
```cpp
for(auto &&[id, type]: entt::resolve()) {
    for(auto&& [propID, property]: type.data())
    {
        // Do whatever with each reflected type property here
    }
}
```
With this out of the way, we have a good starting point!

**For simplicity's sake more details about entt reflection are left out. [CHECK this wiki page for more details](https://github.com/skypjack/entt/wiki/Crash-Course:-runtime-reflection-system)**


# How it works
Now that we covered some of the options I was considering let's get into it!
I decided to go with Entt reflect with our own Imgui integration Because we want to expand this with Lua and serialization.
It's a smarter approach than using the other options out there, As this will keep the codebase cleaner!

## User front end
Let's start talking about how the end user works with the reflection system and how that's set up.

We show examples using `IMPLEMENT_REFLECT_COMPONENT` If you get the idea of how that works, you should be able to understand the other `IMPLEMENT_REFLECT_`( `SYSTEM`, `OBJECT`, `ENUM`, `LEVEL`) types. They work in the same way, except that they have different default functions/properties

### Component header
I will describe everything with the use of the `Example Component`
Its header is defined as follows:
```cpp
//Component.h includes `"Meta/MetaReflect.h"` and some other includes we need for engine reasons
// But there is no Component base class !!
#include "Component.h"
struct ExampleComponent
{
    glm::vec3 someVector;
    std::string someText= "Text value";
    float someFloat = 0.5f;
    REFLECT()
};
```

#### REFLECT macro
```cpp
#ifdef __clang__
#define REFLECT(TYPE)\
    private: static int InitTypeReflect();\
__attribute__((constructor)) static void initialize(void) {\
    InitTypeReflect(); \
};
#else
#define REFLECT(TYPE)\
    private: static int InitTypeReflect();\
    static inline int id = InitTypeReflect();
#endif
```

In the sample shown above I share the implementation details of the REFLECT macro. 
What we do is add a private function to the class that is called before `main()` This allows us to register All types before `main` is even called!
I need to support both `MSVC` and `Clang`. These compilers have different implementations as they don't support a solution that works with both compilers out of the box.

For `MSVC` we have defined a static int variable which causes the function to be invoked before main. More info on what `InitTypeReflect` does will follow.

### Component cpp
For the .cpp file, we have to use the `IMPLEMENT_REFLECT_COMPONENT` as shown below, this will set the entt type to reflect, and create the scoped variable `meta`

Within this function, you can set up entt reflection data as the system describes.
We created another set of macros for the `.prop(<KEY>,<VALUE>)` as an example this is what `PROP_DISPLAYNAME` looks like: `.prop("DisplayNameProperty"_hs, "DisplayName")`.

All those macros are there to make it easier for the gameplay programmer to add properties. When we use macros they show in autocomplete as soon as you type `PROP_`, 
Much easier to work with, you can't make typos. It's worth the little effort it takes.

We also make use of `hidden` properties, Think of a component that is used in the engine, but shouldn't be shown to gameplay programmers.
For instance, we have a `NameComponent` that gives the object a name in the editor and maybe it allows you to search for entity by name.
The user doesn't need to know it's implemented with a separate component, And we have an API that allows you to interact with this without knowing how it's stored.

Below is the .cpp file of the ExampleComponent, demonstrating how to define variables and their properties on an object.
```cpp
#include "MetaReflectImplement.h" // This header will include all the macros shown below.
IMPLEMENT_REFLECT_COMPONENT(ExampleComponent)
{
    meta.data<&ExampleComponent::someVector>("m_SomeVectorNormal"_hs)
        PROP_DISPLAYNAME("SomeVector")
        PROP_DESCRIPTION("This vector is readonly vector display of the colour defined below, as a normalized value!")
        PROP_NORMALIZED
        PROP_READONLY;

    meta.data<&ExampleComponent::someVector>("m_SomeVectorColor"_hs)
        PROP_DISPLAYNAME("SomeVector Color")
        PROP_DESCRIPTION("This vector is drawn as a color!")
        PROP_COLORPICKER;

    meta.data<&ExampleComponent::someText>("m_SomeText"_hs)
        PROP_DISPLAYNAME("TextInput")
        PROP_DESCRIPTION("This is a text input!");

    meta.data<&ExampleComponent::someFloat>("SomeFloat"_hs)
        PROP_DISPLAYNAME("FloatValue")
        PROP_DESCRIPTION("Flaot value that is calmped between 0-1")
        PROP_MINMAX(0.f, 1.f)
        PROP_DRAGSPEED(0.01f); 
}
FINISH_REFLECT()
```

#### IMPLEMENT_REFLECT_COMPONENT
For the sake of the length of this blog, I will skip the full breakdown of the code shown below.

```cpp
template<typename T>
static inline entt::meta_factory<T> ReflectComponent(const std::string& name)
{
    //This is done to make sure Meta is initialized (For DLLs)
    entt::locator<entt::meta_ctx>::reset(GetEngine().GetMetaContext());

    auto factoryT = entt::meta<T>();

    //Entt functions
    //Only enable these functions if there is a default constructor
    if constexpr (std::is_constructible_v<T>) {
        factoryT.func<&entt::registry::emplace<T>>(f_AddComponent);
        factoryT.func<&entt::registry::emplace_or_replace<T>>(f_TryAddComponent);
        factoryT.func<&entt::registry::patch<T>>(f_PatchComponent);
    }

    factoryT.func<&entt::registry::erase<T>>(f_RemoveComponent);
    factoryT.func<&entt::registry::any_of<T>>(f_ContainsComponent);

    //Setup
    factoryT.type(entt::hashed_string{ name.c_str() });
    factoryT.prop(T_ReflectedComponent);
    factoryT.prop(T_DisplayName,name);

    return factoryT;
}
```
The general gist is we call the ReflectComponent from within `Type::InitTypeReflect()`
This sets up the `entt::metaFactory` for the type.
Then we register functionality that is shared between all components.
After that, we pass the factory back to the user who can use it through the `meta` variable.


### The Inspector

Now that we can expose all properties to the reflection system it's time to figure out how to draw UI around it.

We start with we make use of `Entt::Registery::Each` to loop over each entity.
We then figure out if the entity has a component (A combination of looping over all types, and checking if the entity is stored in that container)
We have some added checks to make sure it's reflected and if it should be drawn (Think of the `hidden` property discussed before)


If this all succeeds we start the recursive process of inspecting a type.
[How this works in detail can be seen here](https://github.com/TheDimin/EnttEditor/blob/main/Source/Inspector.cpp)
But the general gist is that we loop over all the properties that have been added to the reflection data of this type.
We then detect if we can "draw" this type, if not we check if the type itself is a container (Vector, Class, SmartPointer, etc.)
We then call the inspect function for the member variable. This allows us to build an inspector tree !!

If we end up finding a variable we get to the fun part!
How do we detect if we can draw a type?
Well, it's in the reflection data !!!

For "generic" types we add the following the there meta type:
```cpp
static auto TypeInspect()
{
    auto type = entt::meta<Type>();
    type.func <&MetaInspectors::Inspect<Type>>(f_Inspect);
    type.func <&Serializers::Serialize<Type>>(f_SaveType);
    type.func <&Serializers::Deserialize<Type>>(f_LoadType);
    return type;
}
#define META_TYPE(Type,...) {[[maybe_unused]] entt::meta_factory<Type> meta = TypeInspect<Type>(); __VA_ARGS__ }
void MetaInit()
{
    entt::locator<entt::meta_ctx>::reset(Perry::GetEngine().GetMetaContext());
    META_TYPE(int);
    META_TYPE(unsigned int);

    META_TYPE(float);
    META_TYPE(bool);
}
```
Because types are now registered we can call these functions from their meta types!
Below is what the `Inspect` function compiles into, Note that the functions shown are `MetaInspect`
In the Inspect function, we cast the value from a `void*` into `T*` and then call `MetaInspect<T>`

```cpp
#define GetOrDefault(KEY,TYPE,DEFAULT) meta.prop(KEY)&&meta.prop(KEY).value() ? meta.prop(KEY).value().cast<TYPE>()  : DEFAULT
    template<typename Type>
    static void MetaInspect(const std::string& name, Type& value, const entt::meta_data& meta)
    {
        //We don't know how to draw this type.
        ImGui::TextColored(ImColor(255, 0, 0), "Missing MetaInspect for: '%s'", meta.type().info().name().data());
    }

    template<>
    static void MetaInspect<int>(const std::string& name, int& value, const entt::meta_data& meta)
    {
        auto speed = GetOrDefault(p_SliderSpeed, float, 1);

        auto min = GetOrDefault(p_ValueMin, int, 0);
        auto max = GetOrDefault(p_ValueMax, int, 0);

        ImGui::DragInt(name.c_str(), &value, speed, min, max);
    }
```
We make one of these `MetaInspect` functions for all the basic types or types that require custom drawing logic.
As you can see this function gets called with the actual type value. If you as a programmer need to implement custom drawing logic, your life should be easy!

### Read-Only
Something that seems missing is the read-only inspector info.

Read only is implemented in a level above Inspect, We have something called pre-post inspect, these allow me to adjust drawing styles globally per property,
and allows us to reuse properties without having to implement them in the `MetaInspect`



# Conclusion
There is a lot that goes into this system. I have tried to show you to most interesting parts, but it's worth looking at the source which can be found here!

The system still requires you to touch different files for generic types, which is an alright compromise, as you don't often have to edit these if you set up the system nicely.

We can even draw arrays of structs without the user having to do anything special what else do we want?
I even integrated LUA using the same concepts. I am not ready to talk about that one yet. But hopefully one day.

<br><br>
Time to back to coding.
- Damian