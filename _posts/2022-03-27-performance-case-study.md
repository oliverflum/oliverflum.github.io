---
title: React Performance Case Study
author: Oliver
date: 2022-03-26 19:00:00 +0010
categories: [SoftwareDevelopment]
tags: [react,typescript]
math: true
mermaid: true
---
# Performance Optimization Case-Study  
When implementing web applications using React, in many cases you will still have to rely on the  
native Web API to respond to events in your application. But when combined with closures and  
React hooks, event listeners create many pitfalls, that might impact performance or break an application altogether and  
that are not obvious or easily understandable. I want to use a fairly basic component I implemented to illustrate some of  
these pitfalls and describe available solutions to avoid issues or alleviate performance degradation.  
## The Component  
The component I will use in this case study is a carousel/slider component that presents multiple options   
vertically lined up and allows the user to scroll through them if there are more options available than the   
slider can fit. To indicate that there are extra options available, two buttons show up on the left and right side  
when the content overflows. They change color when the end of the scrollable area is reached.  
To accommodate users that can't scroll horizontally (e.g. when using a mouse with a scroll wheel), the buttons  
scroll the content in the direction they are pointing to when clicked.   
To allow the color changing behavior, the component will require a scroll event listener that checks whether   
the scrollable area reached the end on either side and then updates the state to change the color.   
In addition, a resize listener on the window is required to make the buttons appear if the slider  
options overflow or disappear if the slider becomes big enough to display all options at the same time.    
The overall component is split into four sub-components. The 'carousel', which is a container for everything.  
The slider which holds the scrollable options and the indicator-buttons. The already mentioned indicator buttons  
and an 'ItemChip' which takes care of the visual presentation of the options. Only the Slider and Indicator components  
contain logic related to event handling, so the others will not be considered much, but are available with the  
rest of the code.  
## Naive Implementation
A naive implementation of that component is displayed below. While it is straightforward and it is easy to understand  
what happens, the naive implementation comes with a number of problems. Most of these can be already demonstrated  
in the Slider component. To keep the article more readable, I will leave out the Scroll Indicator for now as well.
### Slider
```typescript 
import * as React from "react";

import ScrollIndicator, {IndicatorDirection} from "./ScrollIndicator";
import QuickFilterChip from "./ItemChip";

import './Slider.scss'

type SliderProps = {
    items: {title: string} []
}

const Slider: React.FC<SliderProps> = (props:SliderProps) => {
    const valuesRef = React.useRef<HTMLDivElement>(null);
    const containerRef = React.useRef<HTMLDivElement>(null);
    const [showIndicators, setShowIndicators] = React.useState((valuesRef.current  as HTMLDivElement).scrollWidth > (valuesRef.current  as HTMLDivElement).clientWidth);

    window.addEventListener('resize', () => {
        const values = valuesRef.current as HTMLDivElement;
        const scrollable = values.scrollWidth > values.clientWidth;
        setShowIndicators(scrollable);
    });

    return <div className="slider">
        {showIndicators &&
        <ScrollIndicator direction={IndicatorDirection.LEFT} containerRef={containerRef} valuesRef={valuesRef}/>}
        <div ref={containerRef} className="slider__value-container">
            <div ref={valuesRef} className="slider__values">
                {props.items.map(item => 
                    <QuickFilterChip key={item.title} label={item.title}/>)}
            </div>
        </div>
        {showIndicators &&
        <ScrollIndicator direction={IndicatorDirection.RIGHT} containerRef={containerRef} valuesRef={valuesRef}/>}
    </div>;
};

export default Slider;
```
This is a straightforward and simple implementation of the described component. It has a state to keep track of whether or not to   
show the indicators which is initiated to the true/false depending on the size of the elements. It also uses references for the elements  
in the DOM that we want to interact with. Unfortunately it doesn't work. When it's executed, it will cause an error message in the console  
and does not render.  
```typescript
react-refresh-runtime.development.js:316 Uncaught TypeError: Cannot read properties of null (reading 'scrollWidth')
```
This can be prevented by adding question marks after reading from the valuesRef, e.g. 
```typescript
React.useState((valuesRef.current  as HTMLDivElement).scrollWidth => React.useState((valuesRef.current  as HTMLDivElement)?.scrollWidth
```
This successfully masks the error and allows you to pass code review. The component now fails quietly and lived happily ever after. 

## Fixing the Issues

To truly address the underlying issue we'll have to look into the workings of react and understand when code is executed and which values  
it reads from its variables at execution time.

#### 1. Getting Refs Right

The first issue we'll address is the incorrect use of references. References are mutable objects that can be used to store stateful data.  
When passed to a react element in the ref key, the reference will always contain a reference to its current DOM node.  
But because the component is not yet rendered, this node does not exist and the reference value is null. Trying to read properties of this  
null value causes the initial error. Adding the question mark makes the reading evaluate to undefined. Therefor the component seems to work,  
but when the window is small enough for the slider to be scrollable, the indicators won't show. Only after the component rendered the first time,  
everything would work.

=> When using refs, be aware that they are only readable after the component is rendered.

### 2. Using useEffect

To make sure the component already rendered when you read the references, you can use the useEffect hook. This hook is passed a function that is    
executed by react after the component was rendered. We wrap the check of whether the indicators should be shown in a function, so we don't have to replicate  
the code for the initialization and end up with this:
```typescript
const Slider: React.FC<SliderProps> = (props:SliderProps) => {
    const valuesRef = React.useRef<HTMLDivElement>(null);
    const containerRef = React.useRef<HTMLDivElement>(null);
    const [showIndicators, setShowIndicators] = React.useState(false);

    React.useEffect(() => {
        const updateIndicators = () => {
            const values = valuesRef.current as HTMLDivElement;
            const scrollable = values?.scrollWidth > values?.clientWidth;
            setShowIndicators(scrollable);
        }
        updateIndicators();
        window.addEventListener('resize', updateIndicators);
    })

    return <div className="slider">
        {showIndicators &&
        <ScrollIndicator direction={IndicatorDirection.LEFT} containerRef={containerRef} valuesRef={valuesRef}/>}
        <div ref={containerRef} className="slider__value-container">
            <div ref={valuesRef} className="slider__values">
                {props.items.map(item => 
                    <QuickFilterChip key={item.title} label={item.title}/>)}
            </div>
        </div>
        {showIndicators &&
        <ScrollIndicator direction={IndicatorDirection.RIGHT} containerRef={containerRef} valuesRef={valuesRef}/>}
    </div>;
};
```
No more exceptions 🥰 Time to look into improving performance. As of now, this implementation has two major issues.  
Every time the component state changes, we unfortunately run the effect and create a new listener, even though it is independent of  
the state and really stays the same between renders. Even worse, every time the effect is run, we register a 'new' listener  
and they start accumulating over time (To see this you can add a new state in the component that increments or changes  
in some way and log it from the listener).  
So let's address 'even worse' first before looking into 'unfortunately'. It has an easy fix. We have to use useEffect's clean-up.
When returning a function from useEffect, it is executed after the component unmounts or rerenders. So removing the  
event listener in that function will get rid of most trouble:  
```typescript
React.useEffect(() => {
    const updateIndicators = () => {
        const values = valuesRef.current as HTMLDivElement;
        const scrollable = values?.scrollWidth > values?.clientWidth;
        setShowIndicators(scrollable);
    }
    updateIndicators();
    window.addEventListener('resize', updateIndicators);
    return () => {
        window.removeEventListener('resize', updateIndicators)
    }
})
```
Now the listeners don't accumulate anymore. But wouldn't it be neat if we also avoided all the unnecessary instantiations of  
listeners and calls to the browser API? How could you sleep at night if you didn't answer this question with yes?  
Fortunately useEffect allows conditional execution of effects based on an array of dependencies. If one of the dependencies  
changes, the hook fires. Because references are mutable, they don't change between renders. Also the setShowIndicators function  
stays usable after rerendering -> We only have to run the effect once and there for pass it an empty array of dependencies.  
```typescript
React.useEffect(() => {...}, [])
```
There are other means of optimizing performance when using callbacks in react, e.g. the useCallback or useMemo hooks, but for  
I don't see any benefit in using them here, so as far as React goes we're done here. Time for some cake.

=> To execute side effects or code that depends on the component already having rendered, use the **useEffect** hook
=> Don't forget to clean up after your effects to prevent performance degradation over time
=> Only run effects when necessary by providing a list of dependencies

### 3. Filter out the noise
Of course there is more to the world than react and in an honest moment of self reflection, you might ask yourself, do I really  
care for all the events that window resizing causes? Do I need that many?  
If the answer is no, there is some further room to improve performance. Throttling and debouncing are techniques that allow  
you to only call event listeners on a subset of all events that were fired. They are similar but not quite alike; same same  
but different. We will use their [lodash](https://lodash.com/docs/4.17.15) implementations.   
**Debouncing** lets you skip events until a more or less final state is reached. In our example, the listener is called once  
the window resizing stops. And because we only want to hide or show the scroll indicator buttons if the window gets bigger or  
smaller ignoring all the events while resizing is fine. The user can't click those buttons anyway at that time.  
```typescript
React.useEffect(() => {
    const values = valuesRef.current as HTMLDivElement;
    const updateIndicators = () => {
        const scrollable = values.scrollWidth > values.clientWidth;
        setShowIndicators(scrollable);
    }
    updateIndicators();
    const debouncedCheckIndicators = debounce(checkShowIndicators, 100, { leading: false, trailing:true });
    window.addEventListener('resize', debouncedCheckIndicators);
    return () => {
        debouncedCheckIndicators.cancel();
        window.removeEventListener('resize', debouncedCheckIndicators)
    }
}, [])
```
**Throttling** should be used when you do care about what happens between the final states, but only that much. It will allow  
you to only react to events once every time a minimum time period has passed. This can be useful with scrolling, not only saving  
resources and optimizing performance but also making components feel less flakey. The slider component doesn't directly react to scroll  
events and finally the stage is set for the scroll indicator.
```typescript
import * as React from "react";
import { throttle } from "lodash";
import classNames from 'classnames'

import './ScrollIndicator.scss';

const SCROLL_BUFFER = 5;

export enum IndicatorDirection {
    LEFT = "left",
    RIGHT = "right"
};
  
type ScrollDirectionType = `${IndicatorDirection}`;
  

type IndicatorProps = {
    direction: ScrollDirectionType,
    containerRef: React.RefObject<HTMLDivElement>
    valuesRef: React.RefObject<HTMLDivElement>
};

const ScrollIndicator: React.FC<IndicatorProps> = (props:IndicatorProps) => {
    const [active, setActive] = React.useState(false);

    React.useEffect(() => {
        let values = props.valuesRef.current as HTMLDivElement;
        let container = props.containerRef.current as HTMLDivElement;
        const checkActive = () => {
            let shouldBeActive;
            if (props.direction === IndicatorDirection.LEFT) {
                shouldBeActive = container.scrollLeft > SCROLL_BUFFER;
            } else {
                shouldBeActive = container.scrollLeft + SCROLL_BUFFER + container.offsetWidth < values.scrollWidth;
            }
            setActive(shouldBeActive);
        }
        checkActive();
        const throttledCheckActive = throttle(checkActive, 100, {leading: false, trailing: true});
        container.addEventListener('scroll', throttledCheckActive);
        return () => {
            throttledCheckActive.cancel();
            container.removeEventListener('scroll', throttledCheckActive);
        }
    }, []);

    const onClick = (e: React.MouseEvent) => {
        let container = props.containerRef.current as HTMLDivElement;
        if (!active) {
            return 
        }
        let scrollDist = container.clientWidth * 0.75;
        if (props.direction === IndicatorDirection.LEFT) {
            scrollDist = -1 * scrollDist;
        }
        let scrollPos = Math.max(container.scrollLeft + scrollDist, 0);
        container.scrollTo({left: scrollPos, behavior: 'smooth'});
    }

    return <div onClick={onClick} className={classNames("scroll-indicator", {
        "active": active, 
        "left": props.direction === IndicatorDirection.LEFT,
        "right": props.direction === IndicatorDirection.RIGHT
    })}>
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 27.6 100">
	        <polygon points="21.2,50 0,6.5 3.2,0 27.6,50 3.2,100 0,93.5 "/>
        </svg>
    </div>
};

export default ScrollIndicator;
```
The useEffect is a little stuffed here, because scrolling left and right don't share the exact same logic, but it  
illustrates the use of throttle nicely. Only react to events every 100ms and only after that delay has passed  
(falling flank). In both cases, don't forget to cancel the throttled/debounced function to prevent delayed listener  
execution if the component unmounts and maybe take a look at the lodash docs and [this](https://css-tricks.com/debouncing-throttling-explained-examples/) helpful article
by David Corbacho before using those functions. Also keep in mind that calling debounce or throttle is a quite expensive  
call itself. If you have many dependencies in your listener you might lose more than you gain.  


The final component is available [here](https://github.com/oliverflum/diesdasdemos).  
Feel free to (ab)use it. 