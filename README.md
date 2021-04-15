# PS4-Mono

![alt text](https://i.imgur.com/ni7m9No.png)

# Introduction
This will be a short write up of my journey start to finish how I was able to run native mono methods from low level C++. I will be covering how to use the mono functions and the troubles I ran into making invoking some functions possible. As of yet I would still like to spend more time refactoring my mono class and figuring out how to call more methods.

# Researching the PS4 UI
I took some time to look into the playstation's user interface process. I quickly realised it was not like other applications on the system boasting the psm section in its application contents. After finding .dll.sprx and .exe.sprx files in the aforementioned psm folder I took to the internet to see if anyone else had looked into these files.

![alt text](https://i.imgur.com/swVbnIW.png)

To my surprise there had been some research documented. I had found that a github user @SocraticBliss had made a python script that takes the packed sony mono files and extracts them to their respective formats [(Found here)](https://gist.github.com/SocraticBliss/37f913dd29969c11550d3fc6c46e2951). From there I could open the unpacked files in dynspy and take a peak under the hood.

# Investigating the Playstation Highlevel Code
Opening the app.exe now unpacked with the script file mentioned before I could see all of the highlevel code in plain text. There was a lot to unpack here and as I suspected the app.exe was the main mono executable.  

![alt text](https://i.imgur.com/2AUliBM.png)

## A few notable classes of interest I had found:

- The entry point to the application is at the namespace **Sce.Vsh.ShellUI.AppMain** in the class **AppMain**. Here is where I found the main thread loop that the ShellUI runs to do the drawing and other mono calls, this will be important later.
- **Sce.Vsh.ShellUI.AppSystem** Namespace with the class **SystemSoftwareVersionInfo** which holds the string **DisplayVersion** which is the software version shown in the settings menu.
- **Sce.Vsh.ShellUI.Settings.SettingsRoot** Namespace with the class **SettingsRootHandler** which actually holds a bool for **showDebugSettings**. Aswell found in this class is an event for hiding some settings panels one of which is refered to as "sandbox". Could be an interesting patch to be made to show both the debug settings and this other menu.
- **Sce.Vsh.ShellUI.TopMenu** Is the namespace that actually houses the main user interface of the playstation. Also found in this namespace in the class **AreaManager** the method **createDevKitPanel** can be found which I was able to invoke to show the debug panel explained later.

# Invoking Mono from c++

After taking some time to look at the highlevel code I decided the next step would be how could this be manipulated. Through a bit of internet searching I found that mono actually supports manipulating the highlevel code from low level natively. They refer to this as embedded Mono the sdk can be found [(Here)](https://www.mono-project.com/docs/advanced/embedding/) and [(Here)](http://docs.go-mono.com/).

From here it was just a learning curve of how to develop code for high level in the low level. Lucky for me the documentation was very useful and to gain the basics only took the better half of two days. To my surprise aswell the mono classes used on the playstation matched exactly to those refrenced in the sdk.

## Opening Mono Executables and Libraries

In order to open any classes or methods we need to open the executable or library which actually holds the class we are looking for. This can be done once and stored globally or like I do in a class instance to be used later.

### Opening the file
```cpp
bool Open_Assembly()
{
  struct MonoDomain* Root_Domain = mono_get_root_domain();

  if(!Root_Domain)
  {
    error();
    return false;
  }

  mono_thread_attach(Root_Domain);

  struct MonoAssembly* Assembly = mono_domain_assembly_open(Root_Domain, "SomeFile.exe");
  
  if(!Assembly)
  {
    error();
    return false;
  }

  struct MonoImage* Assembly_Image = mono_assembly_get_image(Assembly);

  if(!Assembly_Image)
  {
    error();
    return false;
  }

  return true;
}
```

- `mono_get_root_domain()` Returns the AppDomain which is the initial domain created when the mono runtime is initialized.

- Since we are interacting with a managed code environment we must call `mono_thread_attach()` to signal to the managed code that the thread that we created exists before we interact with any managed object or use any other mono functions.

- `mono_domain_assembly_open()` Loads the contents of **"SomeFile.exe"** into the root domain we opened before. It should be noted that this does not execute the loaded assembly.

- `mono_assembly_get_image()` Gets the Image from the assembly we opened.

## Getting classes and properties

First I wanted to start by figuring out how to grab classes and members from their classes. This would be important to know for later since a lot of classes have instances and in order to call non static functions this would be required.

### Opening a class
```cpp
struct MonoClass*
mono_class_from_name (struct MonoImage *image, const char* name_space, const char *name);

struct MonoClass* OpenClass(const char* Name_Space, const char* Name)
{
  struct MonoClass* Klass = mono_class_from_name(Assembly_Image, Name_Space, Name);

  if(!Klass)
  {
    error()
    return nullptr;
  }

  return Klass;
}

```

- Opening a class is quite simple the `mono_class_from_name()` function will look up our class by name in the name space specified. The assembly image only needs to be opened once so we could use the one we opened before. This function will return zero on failure to find the specified class.

### Getting and setting properties of a class

```cpp
struct MonoProperty*(*mono_class_get_property_from_name)(struct MonoClass *klass, const char *name);
struct MonoMethod*(*mono_property_get_get_method)(struct MonoProperty *prop);
struct MonoMethod*(*mono_property_get_set_method)(struct MonoProperty *prop);
struct MonoObject*(*mono_runtime_invoke)(struct MonoMethod *method, void *obj, void **params, struct MonoObject **exc);
void*(*mono_object_unbox)(struct MonoObject* obj);

MonoObject* Get_Instance(struct MonoClass* klass, const char* Instance)
{
  struct MonoProperty* inst_prop = mono_class_get_property_from_name(klass, Instance);

  if(inst_prop == nullptr)
  {
      MonoLog("Failed to find Instance property \"%s\" in Class \"%s\".", Instance, klass->name);
      return nullptr;
  }

	struct MonoMethod* inst_get_method = mono_property_get_get_method(inst_prop);

  if(inst_get_method == nullptr)
  {
      MonoLog("Failed to find get method for \"%s\" in Class \"%s\".", Instance, klass->name);
      return nullptr;
  }

	struct MonoObject* inst = mono_runtime_invoke(inst_get_method, 0, 0, 0);

  if(inst == nullptr)
  {
      MonoLog("Failed to find get Instance \"%s\" in Class \"%s\".", Instance, klass->name);
      return nullptr;
  }

  return inst;
}

template <typename result>
result Get_Property(struct MonoClass* Klass, const char* Property_Name, MonoObject* Instance)
{
    struct MonoProperty* Prop = mono_class_get_property_from_name(Klass, Property_Name);

    if(Prop == nullptr)
    {
        MonoLog("Property \"%s\" could not be fount on class \"%s\".", Property_Name, Klass->name);
        return (result)nullptr;
    }

    struct MonoMethod* Get_Method = mono_property_get_get_method(Prop);

    if(Get_Method == nullptr)
    {
        MonoLog("Could not find Get Method for \"%s\" in class \"%s\".", Property_Name, Klass->name);
        return (result)nullptr;
    }

    struct MonoObject* Obj = mono_runtime_invoke(Get_Method, Instance, NULL, NULL);

    if(Obj == nullptr)
    {
        MonoLog("Obj refrence was null.");
        return (result)nullptr;
    }

    return *(result*)mono_object_unbox(Obj);
}

template <typename Param>
  void Set_Property(struct MonoClass* Klass, const char* Property_Name, struct MonoObject* Instance, Param Value)
  {
      struct MonoProperty* Prop = mono_class_get_property_from_name(Klass, Property_Name);

      if(Prop == nullptr)
      {
          MonoLog("Property \"%s\" could not be fount on class \"%s\".", Property_Name, Klass->name);
          return;
      }

      struct MonoMethod* Set_Method = mono_property_get_set_method(Prop);

      if(Set_Method == nullptr)
      {
          MonoLog("Could not find Set Method for \"%s\" in class \"%s\".", Property_Name, Klass->name);
          return;
      }

      void* Argsv[] = { &Value };
      mono_runtime_invoke(Set_Method, Instance, Argsv, NULL);
  }
```

This is where things will start to get confusing but fear not the code above can be esily simplified and explained. In mono properties can be stored in an instance and have a set method and a get method. What we are needing to do here is get the set/get method and invoke it in order to interact with the given property.

In C# this would look like this:
```c#
public class MyClass
{
  public string MyString
  {
    get;
    set;
  }
}
```

- `mono_class_get_property_from_name()` Will get our property by name and in our case our property would be `MyString`.
- `mono_property_get_get_method()` Will get the get method for our property we found earlier. Alternatively you could also call the function `mono_property_get_set_method()` which would get the method required to set our property and would be invoked in the same way just with out a return.
- `mono_runtime_invoke` Calling this with our get method as the method and our instance as the obj will return a `MonoObject*`.
- A `MonoObject` must be unboxed in order to get the data from the property we would like. So calling `mono_object_unbox()` and reading our data type from the return pointer will give us the data from the property.

## Example

```cpp
void SetVersionString(const char* str)
{
  struct MonoClass* SystemSoftwareVersionInfo = OpenClass("Sce.Vsh.ShellUI.AppSystem", "SystemSoftwareVersionInfo");

  if(!SystemSoftwareVersionInfo)
  {
    klog("SetVersionString: Failed to open class.\n");
    return;
  }

  struct MonoObject* SystemSoftwareVersionInfo_Instance = Get_Instance(SystemSoftwareVersionInfo, "Instance");

  if(!SystemSoftwareVersionInfo_Instance)
  {
    klog("SetVersionString: Failed to open Instance.\n");
    return;
  }

  struct MonoString* New_String = mono_string_new(Root_Domain, str);

  struct MonoProperty* Prop = mono_class_get_property_from_name(SystemSoftwareVersionInfo, "DisplayVersion");

  if(Prop == nullptr)
  {
      klog("SetVersionString: Could not find property.\n");
      return;
  }

  struct MonoMethod* Set_Method = mono_property_get_set_method(Prop);

  if(Set_Method == nullptr)
  {
      klog("SetVersionString: Could not find set method.\n");
      return;
  }

  mono_runtime_invoke(Set_Method, SystemSoftwareVersionInfo_Instance, New_String, NULL);
}

int Main()
{
  if(Open_Assembly())
    SetVersionString("Cool Version");

  return 0;
}
```

![alt text](https://i.imgur.com/1CvMMRC.png)

# Invoking Methods
```cpp
struct MonoMethod*(*mono_class_get_method_from_name)(struct MonoClass *klass, const char *name, int param_count);

struct MonoObject* Internal_Invoke_Static(struct MonoClass* klass, const char* Method, int Argc, void** Args)
{
    if(klass == nullptr)
    {
        MonoLog("klass was null...");
        return nullptr;
    }

    struct MonoMethod* Function_Method = mono_class_get_method_from_name(klass, Method, Argc);

    if(Function_Method == nullptr)
    {
        MonoLog("Failed to find method \"%s\" in Class \"%s\".", Method, klass->name);
        return nullptr;
    }

    return mono_runtime_invoke(Function_Method, NULL, Args, NULL);
}

template <typename result, typename... Args>
result Invoke_Static(struct MonoClass* klass, const char* Method, Args... args)
{
    void* Argsv[] = { &args... };
    struct MonoObject* Obj = Internal_Invoke_Static(klass, Method, ARRAY_COUNT(Argsv), Argsv);

    if(Obj == nullptr)
    {
        MonoLog("Obj refrence was null.");
        return (result)0;
    }

    return *(result*)mono_object_unbox(Obj);
}
```

This was what I came up with to invoke my first mono method. I figured starting with a static method would be easiest.

- `mono_class_get_method_from_name()` Will search for the method we would like by name in the class we desire returning a `MonoMethod*`.
- `mono_runtime_invoke()` Like when we used this function before we can use it to invoke the method returned by `mono_class_get_method_from_name()` with a null instance. The third paramater being an array of pointers to the variables for the method.
- Like used before we can call the function `mono_object_unbox()` to pull the return value from the `MonoObject` and read the value of our return. Note if our return would be a string we would have to use the function `mono_string_to_utf8()` on the object.

### Example
```cpp
bool IsButtonDown(int Button)
{
    return ShellUI_App->Invoke_Static<bool>("Sce.Vsh.ShellUI.DebugSystem", "KeyMonitorTask", "IsButtonDown", Button);
}

int Main()
{
  for(;;)
  {
    Sleep(1);
    if(IsButtonDown(GamePadButtons::Left | GamePadButtons::Square))
    {
        Notify("Hello from mono function call!! :)");
        Sleep(2000);
    }
  }
}
```

Sucess!! Calling this method returns our expected result when the game pad buttons are pressed.

![alt text](https://i.imgur.com/w03Fahj.png)

## Now the problem
```cpp
struct MonoClass* AreaManager = ShellUI_App->Get_Class("Sce.Vsh.ShellUI.TopMenu", "AreaManager");
ShellUI_App->Invoke(AreaManager, "Instance", "createDevKitPanel");
```

Now I was having fun playing with the functions I could now invoke at my own will. An issue arose though when I decided to see if I could get the devkit panel to draw. 

```
Unhandled Exception: System.AggregateException: Fatal exception was thrown on sub thread. ---> System.InvalidOperationException:  called the method which has to be called on main thread.
  at Sce.PlayStation.HighLevel.UI2.Panel..ctor () [0x00000] in <filename unknown>:0
  at Sce.Vsh.ShellUI.TopMenu.AreaManager.createDevKitPanel () [0x00000] in <filename unknown>:0
   Exception_EndOfInnerExceptionStack
  at Sce.PlayStation.HighLevel.UI2.UISystem.CheckSubThreadFatalException () <0x8204100f0 + 0x000a3> in <filename unknown>:0
  at Sce.PlayStation.HighLevel.UI2.UISystem.Update (System.Action onUpdateInput, System.Action onUpdateAnimations) <0x82040d9a0 + 0x0004a> in <filename unknown>:0
  at Sce.PlayStation.HighLevel.UI2.Application.Update () <0x8204a4fe0 + 0x0011c> in <filename unknown>:0
  at Sce.PlayStation.HighLevel.UI2.Application.Run () <0x8204a4e10 + 0x00083> in <filename unknown>:0
  at Sce.Vsh.ShellUI.AppMain.AppMain.Main (System.String[] args) <0x82ff0df30 + 0x001f4> in <filename unknown>:0
---> (Inner Exception #0) System.InvalidOperationException:  called the method which has to be called on main thread.
  at Sce.PlayStation.HighLevel.UI2.Panel..ctor () [0x00000] in <filename unknown>:0
  at Sce.Vsh.ShellUI.TopMenu.AreaManager.createDevKitPanel () [0x00000] in <filename unknown>:0 <---
```

When invoking this method I observed that it required to be called from the main thread to be executed correctly.

## Hooking the main thread loop
Not all methods are made the same we cant call some methods from any thread we would like, some require them to be called from the main thread. This will require us to hook the main thread loop in the **App.exe**. You may ask your self but where would that be found. Simply enough the entry point of the main application actual holds the main loop.

![alt text](https://i.imgur.com/YZsbjrj.png)
![alt text](https://i.imgur.com/297LlG9.png)

I actually decided to hook the function `UISystem.BeginRender()` since it was an easy enough function to find and would certainly be called from the main thread. 

How do you go about finding the function offset though for the low level code to hook you may ask. The one method I used which is quite simple the method `UISystem.BeginRender()` contains a very specifc numeric value.

![alt text](https://i.imgur.com/MbyMz7s.png)

A 6 followed by 8000 seems simple enough to find. Sure enough a quick search with IDA nets us very few results to narrow down. This is only one method and just a quick and dirty version I reckon dumping the loaded code from memory could make this easier though I have not tested.

![alt text](https://i.imgur.com/ZteFs3z.png)

![alt text](https://i.imgur.com/OMZOIOP.png)

# Sucess we can now invoke our method
```cpp
void BeginRender()
{
    static bool done_once = false;
    
    if(done_once == false)
    {
        MonoClass* AreaManager = ShellUI_App->Get_Class("Sce.Vsh.ShellUI.TopMenu", "AreaManager");
        ShellUI_App->Invoke(AreaManager, "Instance", "createDevKitPanel");
        ShellUI_App->Invoke(AreaManager, "Instance", "createExhibitionPanel");
        ShellUI_App->Invoke(AreaManager, "Instance", "createFactoryPanel");

        done_once = true;
    }
    
    BeginRenderStub();
}
```

![alt text](https://i.imgur.com/ni7m9No.png)

# Conclusion
Now after two days I had managed to complete my goal of invoking both static and instanced mono methods. I was now able to call mono methods that were required to be called on the main thread. I now have set my sights on my next goals. 

- Refactoring my mono classes as they have a few issues at the moment mainly passing in pointers as variables.
- Invoking constructor methods to create my own instances of classes.
- Appending my own Widget to the TopMenu to begin work of drawing my own menu.
- Making modifications to the current playstation user interface ie. Adding my own menus.

For now I am happy with my progress and I am very happy to share my jorney with you all. I hope this has been an interesting read as well as informative. Hopefully it can serve as inspiration to others to learn more as well to contribute to the community.
