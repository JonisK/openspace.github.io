---
title: Time in OpenSpace
layout: default
parent: User Tutorials
grand_parent: Tutorials

has_children: false
has_toc: false
nav_order: 4
---

# Time in OpenSpace
{: .no_toc }

This section will cover the various aspects of manipulating time in OpenSpace.

1. TOC
{:toc}

## The Date/Time of the Simulation
 - Upon launch of the default scene, the simulation will be set to one day behind the current time.
 - The date of the simulation can be change to any date.
 - Solar system positional information is only available between 1850 and 2150.

## Changing the Rate of Time Change in the Simulation
 - By default the simulation will progress in realtime.
 - The rate at which the simulation progress can be changed.
 - You can progress forwards or backwards in time at any speed.
 - Time can be paused.
 - See [Using the menus: Time](/docs/tutorials/users/menustime) for details on controling the rate.

## Interpolate vs Instant Time Change
 - When changing the date, the rate of time change, or pausing time, the change will be interpolated as to not 'jump' the camera.
 - The number of seconds it will take to interpolate can be set under Settings->Time Manager where there are individual settings for the Time change, Rate of time change, and pusing.
 - Various UI elements, lua scripts, and keyboard shortcuts will let you make these changes 'instantly' without the interpolation. 

## Keyboard Shortcuts for Time
 - Space bar will pause time (slowing it down to a pause over 0.5 seconds)
 - Shift + Space bar will pause time immediatly
 - The left arrow key will increase the rate of time (increasing over 0.5 seconds).
 - The right arrow key will decrease the rate of time (decreasing over 0.5 seconds).
 - Shift + the arrow keys will increase/decrease the rate of time immediatly.
 - The number keys will set increasing specific rates of time.
 - Alt + number keys will set a decreasing negative rates of time.


## Video

 See the next tutorial [Using the menus: Time](/docs/tutorials/users/menustime) for a video detailing using the time menu.


---
<span class="v-align-middle">
[Navigation - Focus](/docs/tutorials/users/navigationfocus){: .btn .btn-purple }
</span>
<span class="fs-6"><-- Previous |</span>
<span class="fs-6">Next -->  </span>
<span class="v-align-middle">
[Using the menus: Time](/docs/tutorials/users/menustime){: .btn .btn-purple }
</span>
