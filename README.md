# mql4-lib

MQL4 Foundation Library For Professional Developers

* [1. Introduction](#introduction)
* [2. Installation](#installation)
* [3. Usage](#usage)
  * [3.1 Basic Programs](#basic-programs)
  * [3.2 Runtime Controlled Indicators and Indicator Drivers](#runtime-controlled-indicators-and-indicator-drivers)
  * [3.3 Collections](#collections)
  * [3.4 Maps](#maps)
  * [3.5 Asynchronous Events](#asynchronous-events)
  * [3.6 File System and IO](#file-system-and-io)

## Introduction

MQL4 programming language provided by MetaQuotes is a very limited version of
C++, and its standard library is a clone of the (ugly) MFC, both of which I am
very uncomfortable with. Most MQL4 programs have not adapted to the MQL5 (Object
Oriented) style yet, let alone reuable and elegant component based design and
programming.

mql4-lib is a simple library that tries to make MQL4 programming pleasant with a
more object oriented approach and a coding style like Java, and encourages
writing reusable components. This library has the ambition to become the de
facto Foundation Library for MQL4.

## Installation

Just copy the library to your MQL4 Data Folder's `Include` directory, with the
root directory name of your choice, for example:
`<MQL4Data>\Include\MQL4\<mql4-lib content>`.

It is recommened that you use the lastest version MetaTrader4, as many features
are not available in older versions.

## Usage

The library is in its early stage. However, most components are pretty stable
and can be used in production. Here are the main components:

1. `Lang` directory contains modules that enhance the MQL4 language
2. `Collection` directory contains useful collection types
3. `Charts` directory contains several chart types and common chart tools
4. `Trade` directory contains useful abstractions for trading
5. `History` directory contains useful abstractions for history data
6. `Utils` directory contains various utilities
7. `UI` chart objects and UI controls (in progress)
8. `OpenCL` brings OpenCL support to MT4 (in progress)

### Basic Programs

In `Lang`, I abstract three Program types (Script, Indicator, and Expert
 Advisor) to three base classes that you can inherit.

Basically, you write your program in a reusable class, and when you want to use
them as standalone executables, you use macros to declare them.

The macro distinguish between programs with and without input parameters. Here
is a simple script without any input parameter:

```c++
#include <MQL4/Lang/Script.mqh>
class MyScript: public Script
{
public:
  // OnStart is now main
  void main() {Print("Hello");}
};

// declare it: notice second parameter indicates the script has no input
DECLARE_SCRIPT(MyScript,false)
```

Here is another example, this time an Expert Advisor with input parameters:

```c++
#include <MQL4/Lang/ExpertAdvisor.mqh>

class MyEaParam: public AppParam
{
  ObjectAttr(string,eaName,EaName);
  ObjectAttr(double,baseLot,BaseLot);
public:
  // optionally override `check` method to validate paramters
  // this method will be called before initialization of EA
  // if this method returns false, then INIT_INCORRECT_PARAMETERS will
  // be returned
  // bool check(void) {return true;}
};

class MyEa: public ExpertAdvisor
{
private:
  MyEaParam *m_param;
public:
       MyEa(MyEaParam *param)
       :m_param(param)
      {
         // Initialize EA in the constructor instead of OnInit;
         // If failed, you call fail(message, returnCode)
         // both paramters of `fail` is optional, with default return code INIT_FAIL
         // if you don't call `fail`, the default return code is INIT_SUCCESS;
      }
      ~MyEa()
      {
         // Deinitialize EA in the destructor instead of OnDeinit
         // getDeinitReason() to get deinitialization reason
      }
  // OnTick is now main
  void main() {Print("Hello from " + m_param.getEaName());}
};

// The code before this line can be put in a separate mqh header

// We use macros to declare inputs
// Notice the trailing semicolon at the end of each INPUT, it is needed
// support custom display name because of some unknown rules from MetaQuotes
BEGIN_INPUT(MyEaParam)
  INPUT(string,EaName,"My EA"); // EA Name (Custom display name is supported)
  INPUT(double,BaseLot,0.1);    // Base Lot
END_INPUT

DECLARE_EA(MyEa,true)  // true to indicate it has parameters
```

The `ObjectAttr` macro declares standard get/set methods for a class. Just
follow the Java Beans(TM) convention.

I used some macro tricks to work around limits of MQL4. I will document the
library in detail when I have the time.

With this approach, you can write reusable EAs, Scripts, or Indicators. You do
not need to worry about the OnInit, OnDeinit, OnStart, OnTick, OnCalculate, etc.
You never use a input parameter directly in your EA. You can write a base EA,
and extend it easily.

### Runtime Controlled Indicators and Indicator Drivers

When you create an indicator with this lib, it can be used as both standalone
(controlled by the Terminal runtime) or driven by your programs. This is a
powerful concept in that you can write your indicator once, and use it in your
EA and Script, or as a standalone Indicator. You can use the indicator in normal
time series chart, or let it driven by a RenkoIndicatorDriver.

Let me show you with an example Indicator `DeMarker`. First we define the common
reusable Indicator module in a header file `DeMarker.mqh`

```MQL5
//+------------------------------------------------------------------+
//|                                                     DeMarker.mqh |
//|                                          Copyright 2017, Li Ding |
//|                                            dingmaotu@hotmail.com |
//+------------------------------------------------------------------+
#property strict

#include <MQL4/Lang/Mql.mqh>
#include <MQL4/Lang/Indicator.mqh>
#include <MovingAverages.mqh>
//+------------------------------------------------------------------+
//| Indicator Input                                                  |
//+------------------------------------------------------------------+
class DeMarkerParam: public AppParam
  {
   ObjectAttr(int,AvgPeriod,AvgPeriod); // SMA Period
  };
//+------------------------------------------------------------------+
//| DeMarker                                                         |
//+------------------------------------------------------------------+
class DeMarker: public Indicator
  {
private:
   int               m_period;
protected:
   double            ExtMainBuffer[];
   double            ExtMaxBuffer[];
   double            ExtMinBuffer[];
public:

   //--- this provides time series like access to the indicator buffer
   double            operator[](const int index) {return ExtMainBuffer[ArraySize(ExtMainBuffer)-index-1];}

                     DeMarker(DeMarkerParam *param)
   :m_period(param.getAvgPeriod())
     {
      //--- runtime controlled means that it is used as a standalone Indicator controlled by the Terminal
      //--- isRuntimeControlled() is a method of common parent class `App`
      if(isRuntimeControlled())
        {
         //--- for standalone indicators we set some options for visual appearance
         string short_name;
         //--- indicator lines
         SetIndexStyle(0,DRAW_LINE);
         SetIndexBuffer(0,ExtMainBuffer);
         //--- name for DataWindow and indicator subwindow label
         short_name="DeM("+IntegerToString(m_period)+")";
         IndicatorShortName(short_name);
         SetIndexLabel(0,short_name);
         //---
         SetIndexDrawBegin(0,m_period);
        }
     }

   int               main(const int total,
                          const int prev,
                          const datetime &time[],
                          const double &open[],
                          const double &high[],
                          const double &low[],
                          const double &close[],
                          const long &tickVolume[],
                          const long &volume[],
                          const int &spread[])
     {
      //--- check for bars count
      if(total<m_period)
         return(0);
      if(isRuntimeControlled())
        {
         //--- runtime controlled buffer is auto extended and is by default time series like
         ArraySetAsSeries(ExtMainBuffer,false);
        }
      else
        {
         //--- driven by yourself and thus the need to resize the main buffer
         if(prev!=total)
           {
            ArrayResize(ExtMainBuffer,total,100);
           }
        }
      if(prev!=total)
        {
         ArrayResize(ExtMaxBuffer,total,100);
         ArrayResize(ExtMinBuffer,total,100);
        }
      ArraySetAsSeries(low,false);
      ArraySetAsSeries(high,false);

      int begin=(prev==total)?prev-1:prev;

      for(int i=begin; i<total; i++)
        {
         if(i==0) {ExtMaxBuffer[i]=0.0;ExtMinBuffer[i]=0.0;continue;}
         if(high[i]>high[i-1]) ExtMaxBuffer[i]=high[i]-high[i-1];
         else ExtMaxBuffer[i]=0.0;

         if(low[i]<low[i-1]) ExtMinBuffer[i]=low[i-1]-low[i];
         else ExtMinBuffer[i]=0.0;
        }
      for(int i=begin; i<total; i++)
        {
         if(i<m_period) {ExtMainBuffer[i]=0.0;continue;}
         double smaMax=SimpleMA(i,m_period,ExtMaxBuffer);
         double smaMin=SimpleMA(i,m_period,ExtMinBuffer);
         ExtMainBuffer[i]=smaMax/(smaMax+smaMin);
        }

      //--- OnCalculate done. Return new prev.
      return(total);
     }
  };
```

If you want to use this as a standalone Indicator, create `DeMarker.mq4`:

```MQL5
//+------------------------------------------------------------------+
//|                                                     DeMarker.mq4 |
//|                                          Copyright 2017, Li Ding |
//|                                            dingmaotu@hotmail.com |
//+------------------------------------------------------------------+
#property copyright "Copyright 2017, Li Ding"
#property link      "dingmaotu@hotmail.com"
#property version   "1.00"
#property strict

#property indicator_separate_window
#property indicator_minimum    0
#property indicator_maximum    1.0
#property indicator_buffers    1
#property indicator_color1     LightSeaGreen
#property indicator_level1     0.3
#property indicator_level2     0.7
#property indicator_levelcolor clrSilver
#property indicator_levelstyle STYLE_DOT

#include <Indicators/DeMarker.mqh>
//--- input parameters
BEGIN_INPUT(DeMarkerParam)
   INPUT(int,AvgPeriod,14); // Averaging Period
END_INPUT

DECLARE_INDICATOR(DeMarker,true);
```

Or if you want to use it in your EA, driven by a Renko chart:
```MQL5
//--- code snippets for using an indicator with IndicatorDriver

//--- OnInit
//--- create the indicator manually, its runtime controlled flag is false by default
   DeMarkerParam param;
   param.setAvgPeriod(14);
   deMarker=new DeMarker(GetPointer(fp));

//--- indicator driver is a container for indicators
   IndicatorDriver *driver=new IndicatorDriver; 
//--- add indicator to the driver: you can add multiple indicators
   driver.add(deMarker);
//--- create the real driver that provide history data
   renkoDriver = new RenkoIndicatorDriver(300,driver);
   
//--- OnTick
   renkoDriver.update(Close[0]);
   
//--- after update, all indicators attached to the IndicatorDriver will be updated
//--- access DeMarker
   double value = deMarker[0];
   
//--- OnDeinit
//--- need to release resources
   delete deMarker;
   delete driver;
   delete renkoDriver;
```

### Collections

In advanced MQL4 programs, you have to use more sophisticated collection types
for your order management.

It is planned to add common collection types to the lib, including lists, hash
maps, trees, and others.

Currently there are two list types:

1. Collection/LinkedList is a Linked List implementation
2. Collection/Vector is an array based implementation

First I'd like to point out some extremely useful undocumented MQL4/5 features:

1. class templates(!)
2. typedef function pointers(!)
3. template function overloading
4. union type

Though inheriting multiple interfaces is not possible now, I think this will be
possible in the future.
 
Among these features, `class template` is the most important because we can
greatly simplify Collection code. These features are used by MetaQuotes to port
.Net Regular Expression Library to MQL.

With class templates and inheritance, I implemented a hierarchy:

    Iterable -> Collection -> LinkeList and Vector

The general usage is as follows:

```c++
LinkedList<Order*> orderList; // linked list based implementation, faster insert/remove
LinkedList<int> intList; //  yes it supports primary types as well
Vector<Order*>; orderVector // array based implementation, faster random access
Vector<int> intVector;
```

To iterate through a collection, use its iterator, as iterators know what is the
most efficient way to iterating.

Threre are alao two macros for iteration: `foreach` and `foreachv`. You can
`break` and `return` in the loop without worrying about resource leaks because
we use `Iter` RAII class to wrap the iterator pointer.

Here is a simple example:

```
//+------------------------------------------------------------------+
//|                                                TestOrderPool.mq4 |
//+------------------------------------------------------------------+
#property copyright "Copyright 2016-2017, Li Ding"
#property link      "dingmaotu@hotmail.com"
#property version   "1.00"
#property strict

#include <MQL4/Trade/Order.mqh>
#include <MQL4/Trade/OrderPool.mqh>
#include <MQL4/Collection/LinkedList.mqh>

// for simplicity, I will not use the Lang/Script class
void OnStart()
  {
   OrderList<Order*> list;
   int total= TradingPool::total();
   for(int i=0; i<total; i++)
     {
      if(TradingPool::select(i))
        {
         OrderPrint(); // to compare with Order.toString
         list.push(new Order());
        }
     }

   PrintFormat("There are %d orders. ",list.size());

   //--- Iter RAII class
   for(Iter<Order*> it(list); !it.end(); it.next())
     {
      Order*o=it.current();
      Print(o.toString());
     }

   //--- foreach macro: use it as the iterator variable
   foreach(Order*,list)
     {
      Order*o=it.current();
      Print(o.toString());
     }

   //--- foreachv macro: declare element varaible o in the second parameter
   foreachv(Order*,o,list)
     Print(o.toString());

  }
//+------------------------------------------------------------------+
```

### Maps

Map (or dictionary) is extremely important for any non-trivial programs.
Mql4-lib impelements an efficient Hash map using Murmur3 string hash. The
implementation follows the CPython3 hash and it preserves insertion order.

You can use any builtin type as key, including any class pointer. These types
has their hash functions implemented in `Lang/Hash` module.

The HashMap interface is very simple. Below is a simple example counting words
of the famous opera `Hamlet`:

```c++
#include <MQL4/Lang/Script.mqh>
#include <MQL4/Collection/HashMap.mqh>
#include <MQL4/Utils/File.mqh>

class CountHamletWords: public Script
  {
public:
   void              main()
     {
      TextFile txt("hamlet.txt", FILE_READ);
      if(txt.valid())
        {
         HashMap<string,int>wordCount;
         while(!txt.end() && !IsStopped())
           {
            string line= txt.readLine();
            string words[];
            StringSplit(line,' ',words);
            int len=ArraySize(words);
            if(len>0)
              {
               for(int i=0; i<len; i++)
                 {
                  int newCount=0;
                  if(!wordCount.contains(words[i]))
                     newCount=1;
                  else
                     newCount=wordCount[words[i]]+1;
                  wordCount.set(words[i],newCount);
                 }
              }
           }
         Print("Total words: ",wordCount.size());
         //--- you can use the foreachm macro to iterate a map
         foreachm(string,word,int,count,wordCount)
         {
            PrintFormat("%s: %d",word,count);
         }
        }
     }
  };
DECLARE_SCRIPT(CountHamletWords,false)
```

  
### Asynchronous Events

The `Lang/Event` module provides a way to send custom events from outside the
MetaTrader terminal runtime, like from a DLL.

You need to call the `PostMessage/PostThreadMessage` funtion, and pass
parameters as encoded in the same algorithm with `EncodeKeydownMessage`. Then
any program that derives from EventApp can process this message from its
`onAppEvent` event handler:

```
   #include <WinUser32.mqh>
   #include <MQL4/Lang/Event.mqh>
   bool SendAppEvent(int hwnd,ushort event,uint param)
     {
      int wparam, lparam;
      EncodeKeydownMessage(event, param, wparam, lparam);
      return PostMessageW(hwnd,WM_KEYDOWN,wparam, lparam) != 0;
     }
```

The mechanism uses a custom WM_KEYDOWN message to trigger the OnChartEvent. In
`OnChartEvent` handler, `EventApp` checks if KeyDown event is actually a custom
app event from another source (not a real key down). If it is, then `EventApp`
calls its `onAppEvent` method.

This mechnism has certain limitations: the parameter is only an integer (32bit),
due to how WM_KEYDOWN is processed in MetaTrader terminal. And this solution may
not work in 64bit MetaTrader5.

Despite the limitations, this literally liberates you from the MetaTrader jail:
you can send in events any time and let mt4 program process it, without polling
in OnTimer, or creating pipe/sockets in OnTick, which is the way most API
wrappers work.

Using OnTimer is not a good idea. First it can not receive any parameters from
the MQL side. You at least needs an identifier for the event. Second, WM_TIMER
events are very crowded in the main thread. Even on weekends where there are no
data coming in, WM_TIMER is constantly sent to the main thread. This makes more
instructions executed to decide if it is a valid event for the program.

*WARNING*: This is a temporary solution. The best way to handle asynchronous
events is to find out how ChartEventCustom is implemented and implement that in
C/C++, which is extremely hard as it is not implemented by win32 messages, and
you can not look into it because of very strong anti-debugging measures.

Inside MetaTrader terminal, you better use ChartEventCustom to send custom
events.

### File System and IO 

MQL file functions by design directly operate on three types of files: Binary,
Text, and CSV. To me, these types of files are supposed to form a layered
relationship: CSV is a specialized Text, and Text specialized Binary (with
encoding/decoding of text). But the functions are NOT designed this way, rather
as a tangled mess by allowing various functions to operate on different types of
files. For example, `FileReadString` behavior is totally different besed on what
kind of file it's opearting: for Binary the unicode bytes (UTF-16 LE) are read
with specified length, for Text the entire line is read (is FILE_ANSI flag is
set the text is decoded based on codepage), and for CSV only a string field is
read. I don't like this design, neither I have the energy and time to
reimplement text encoding/decoding and type serializing/deserializing.

So I wrote a `Utils/File` module, wrapping all file functions with a much
cleaner interface, but without changing the whole design. There are five
classes: `File` is a base class but you can not instantiate it; `BinaryFile`,
`TextFile`, and `CsvFile` are the subclasses which are what you use in your
code; and there is an interesting class `FileIterator` which impelemented
standard `Iterator` interface, and you can use the same technique to iterate
through directory files.

Here is a example for TextFile and CsvFile:

```c++
#include <MQL4/Utils/File.mqh>

void OnStart()
  {
   File::createFolder("TestFileApi");

   TextFile txt("TestFileApi\\MyText.txt",FILE_WRITE);
   txt.writeLine("你好，世界。");
   txt.writeLine("Hello world.");

//--- reopen closes the current file handle first
   txt.reopen("TestFileApi\\MyText.txt",FILE_READ);
   while(!txt.end())
     {
      Print(txt.readLine());
     }

   CsvFile csv("TestFileApi\\MyCsv.csv",FILE_WRITE);
//--- write whole line as a text file
   csv.writeLine("This,is one,CSV,file");

//--- write fields one by one
   csv.writeString("这是");
   csv.writeDelimiter();
   csv.writeInteger(1);
   csv.writeDelimiter();
   csv.writeBool(true);
   csv.writeDelimiter();
   csv.writeString("CSV");
   csv.writeNewline();

   csv.reopen("TestFileApi\\MyCsv.csv",FILE_READ);
   for(int i=1; !csv.end(); i++)
     {
      Print("Line ",i);
      //--- notice that you SHALL NOT directly use while(!csv.isLineEnding()) here
      //--- or you will run into a infinite loop
      do
        {
         Print("Field: ",csv.readString());
        }
      while(!csv.isLineEnding());
     }
```

And here is an example for `FileIterator`:

```c++
#include <MQL4/Utils/File.mqh>

int OnStart()
{
   for(FileIterator it("*"); !it.end(); it.next())
     {
      string name=it.current();
      if(File::isDirectory(name))
        {
         Print("Directory: ",name);
        }
      else
        {
         Print("File: ",name);
        }
     }
}
```

Or you can go fancy with the powerful `foreachfile` macro:

```c++
#include <MQL4/Utils/File.mqh>

int OnStart()
{
//--- first parameter is the local variable *name* for current file name
//--- second parameter is the filter pattern string
   foreachfile(name,"*")
     {
      if(File::isDirectory(name))
         Print("Directory: ",name);
      else
         Print("File: ",name);
     }
}
```
