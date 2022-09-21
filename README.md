# MantaRay

### A ray tracing research project.

###### This document has been edited for GitHub. 

![](https://pre00.deviantart.net/03ad/th/pre/i/2014/291/6/0/manta_ray_by_squid_coffee-d83d7d0.png)

Abstract
---

Ray tracing allows for the rendering of very high quality imagery by shooting rays through the camera into a scene and letting these intersect and bounce around while staying as close to the laws of physics as possible. The paths of these rays are chosen carefully to make to sure that only those rays that will actually contribute to the final render of the scene are computed. Compared to other approaches to the rendering equation ray tracing excels at the accurate display of spheres and reflections. However, the algorithm does come at a price. Its exponential computation cost greatly limits its use in real-time applications such as video games. Despite this, due to its incredible simulation of real light, it is an amazing method for rendering film and TV as these can be processed slowly ahead of time.

### Acknowledgements

I would like to extend special thanks to my supervisor Dr. Richard Overill.

Contents
---

#### Introductory

Motivation

Objectives

Platform

#### Fundamentals

Ray Tracing

Intersecting a sphere

Precision problems

Intersecting a box

Bounding volumes

Intersecting a plane

Branching

#### Shading

Light Rays

Shadow Rays

Reflection

Phong

#### Addendum

Evaluation

Conclusions

Bibliography and References

User Guide

Introductory
---

#### Motivation

The desire for ultra realism in our digital lives is exceptional. Even now people are already trying to surpass ultra HD. We humans like to create, we like to experience. And what experience is more immersive than true realism. What if we could create our own pseudo realism. The ultimate creation. It won’t ever be real real, but our brains do not need to know that. With Virtual Reality as the new kid on the block and quantum computing slowly starting to have its name carried on the distant winds it is time to dust off the research that was done on ray tracing 30 to 40 years ago. As within a couple decades we might very well have the computational power required to create realities beyond our wildest imaginations.

#### Objectives

The objective is to design and implement a fundamental ray tracer. Starting off with getting to know the mathematical algorithms and formulas upon which ray tracing is build. Such as intersecting geometric primitives, illuminating and shading points in the scene, and reflecting rays off of appropriate surfaces. Ultimately a program is to be created that is able to show the power of the ray tracing algorithm.

#### Platform

The ray tracer will primarily be developed in OpenCL. As the ray tracing algorithm doesn’t really apply any traditional form of rendering we can repurpose the GPU as a computation unit and exploit its power. The OpenCL kernel will be compiled and launched from a Java application. This application will also take care of copying any data from the host machine to the OpenCL device, as well as reading back any relevant data that was computed by the kernel.

The OpenCL implementation will be complemented by a small amount of OpenGL code to create an application window and display the generated render to the user.

Fundamentals
---

At the heart of ray tracing lie the mathematical algorithms involving the intersection of rays with a collection of primitive shapes. A ray is defined as a line of infinite length in a particular direction with a fixed starting point. Primitive shapes include spheres, cubes and boxes, planes and polygons, pyramids, cylinders and toroids (think donuts), and spline curves.

In the following sections a ray will be defined as R(t) = R0 + t * Rd, where t > 0. Let R0 be the origin of the ray, let Rd be its normalised direction, and let t be a scalar to define any point along the ray.

![](https://lh4.googleusercontent.com/PLiBVjBShe-n9nqO0HqpEhypf25guSRCRFOw7HX-WF9xuvD9wVZfsTmk2sEqiuSTswkgC2nhey5bWLWWdFSHwZ5BdAYwiVS0k8Q9B_kHZCxVThATG1AhyXrW2hRkWhrbOXIsCr6n)
###### Figure 2.1 A ray struct in OpenCL and a ray in terms of t in three-dimensional space.

We will first be taking a look at how to check whether, and if so where, a ray intersects a sphere and a box. In addition to being shapes that can be rendered, these two primitives are often used as bounding volumes. Bounding volumes are an incredibly powerful optimisation method as they exponentially reduce the amount of calculations needed to render increasingly more complex scenes.

#### Intersecting a sphere

Contrary to other rendering methods the accurate rendering of spheres comes for free with ray tracing. We are going to look at how to ray trace a sphere geometrically as this approach is strictly more efficient than calculating the intersection algebraically. In addition, it allows for exiting the calculation sooner if there happens to be no intersection.

![](https://lh5.googleusercontent.com/6cviF7sf-mOjDJP_L70VNli6pmH8ePzjVL0cR2N-7x9TS_VSFiAg4zcK9xIpFAPUAAhsU6QkVVXTBjPe61Nplnuy2WI2U8WJtVchFgzeK3BSTMm23iJBfrvx9FVOE4lNNgIpwcAH)
###### Figure 2.2 A sphere struct in OpenCL. The squared radius is cached on creation to avoid having to calculate it multiple times in the future.

To start off the calculation the first condition to look at is whether the origin of the ray lies within the sphere. In computer graphics it is common for shapes to only be visible when their surface normal is pointing towards the camera (or intersecting ray).

![](https://lh4.googleusercontent.com/zOnXK-D3aGEia44H_yEgzDBkQSs0u0MbPdQQu5RU0r4s_vsaeHciRVpWQjD22o9ScF_P_m7pdvTl36JYrnCL0UEDsjCv20f3i-6JSFCu2sJT2KD2owe0f84Up5h5D5ITt1kpespC)
###### Figure 2.3 Testing whether a ray’s origin is inside or on a sphere.

The normals on a regular sphere will always point away from its center. This means that if the origin of a ray is inside the sphere, regardless of exactly where it intersects the surface of the sphere, we can be sure that the normal at the point of intersection points away from it. 

The normal of any point on the surface of a sphere can be found by subtracting the center of the sphere from the point in question and normalising the result.

In addition to considering intersections where the origin of the ray is inside the sphere illegal, neither should the ray originate on an actual surface point of the sphere. This introduces complications with reflection rays which will be looked at in more detail in ‘Precision problems’.

![](https://lh3.googleusercontent.com/m7gRxVwZxPwww1Ov-51hbd9DDiuLN7wDowM-4BBKpt1qEL9aU3llPawZw988WwaJEzOBeqxzlK8wM9s1nLQfWsolMDwDNisyHOosBjbpNOyReuyl56wa7qVo0A7WDrfcjIzra0VW)
###### Figure 2.4 A visualisation of the closest approach in terms of t for rays pointing towards the sphere as well as pointing away from it.

The remainder of the computation can only be reached by rays which have their origin outside of the volume of the sphere, which we will exploit later. The next step is to make sure the ray is not pointing away from the sphere (i.e. the center of the sphere is behind the ray). This can be checked by calculating the position of the closest approach of the ray to the center of the sphere along the ray in terms of t. Where tca is positive the center of the sphere lies in front of the ray (Figure 2.4 on the left), and subsequently if tca is negative the center of the sphere lies behind the ray (Figure 2.4 on the right).

![](https://lh6.googleusercontent.com/ag0vRViACkHBCZrNjYDF9wlynhAEMD5Rkzp6vEatOFZ_ISb8XgcoQAQq_Pajdd2tvKBdC50WDUvnJxgakYQmfhJEfWlvln_Q3W6Y6faW-2bic8lIGmKIsFDJ_khgsJ8RtfXgdh5z)
###### Figure 2.5 Testing whether the center of a sphere lies behind or in front of the origin of a ray.

The closest approach can be calculated along the ray in terms of t by taking the dot product of the distance that was calculated in the previous step with the normalised direction of the ray (imagine the vector going from R0 to Sc which is then projected back onto the ray). Considering that all rays at this point in the computation have their origin outside of the volume of the sphere; as tca approaches 0 one should be able to see that the origin of the ray is pushed to the side of the sphere rendering it unable to intersect the sphere when tca equals 0.

![](https://lh5.googleusercontent.com/GgkkUq_oK-kgjam-3qJmKOmBRKfEGDBfc2njP0JfnZyos-c9J7Wv5KGx3QTYE9Lb710picP2ED0MwXcXPB1wD0U9BIWLNl5xPs2xa-AZgIuu46qjFDRNzGTsSBJhgjrB3CJhYsZR)
###### Figure 2.6 The half chord distance is highlighted in orange.

Finally the half chord distance, the distance from the closest approach to the surface of the sphere, is computed. It follows that if an intersection does occur between this ray and sphere the value for t can be found by subtracting the half chord distance from the closest approach.

![](https://lh4.googleusercontent.com/aJUEPDlrCK07UPfSLmAC81yHonS-2eg5LME3OeTNmDJuhB-pcUMESTL0DR2mF-_26ng-1GFgn3tkq-eg6n7ZCALPsfWyPRb674aOQKfFne0m4ekDzYhU5VMXU0BCyF0jxDwAxXt9)
###### Figure 2.7 Final test of whether a ray misses a sphere.

One last check is performed to filter out rays where the half chord distance squared is negative as these rays miss the sphere.

![](https://lh3.googleusercontent.com/E0CWmgJcQ-KG7G5pbDi5Vx8KS_QIJsZZWKP5MXKSzvk03iI4_KQcPZEzJ_6CtD2eG08vaxBPh4ajJ2gPXWZYaMC3nbJbsCc8m-LhO6tey-1J0zY4Z0z6TYOi13yz9YQt4fjeCe7L)
###### Figure 2.8 The point of intersection is found and the value for t is stored on the ray.

Everything required to calculate the point of intersection has been found and any chance that the ray misses the sphere has been eliminated. However, as we are not entirely sure yet whether this sphere will also end up being the closest object to the camera we might not want to spend the resources yet to determine the actual point of intersection. For now the value t and the material of the object the ray intersected is all we need. This value for t can be compared to the t value of any future intersections to determine the closest intersected object. After all objects in the scene have been tested for intersection the final t value and material on the ray are used to colour the pixel this ray belongs to.

#### Precision problems

Doing floating-point calculations is like moving piles of sand around. Every time you move a pile you lose a little sand and pick up a little dirt [4]. Whenever the closest shape to the viewport that this ray intersects has been found there is a floating point precision problem that needs to be addressed.

![](https://lh6.googleusercontent.com/76TjkTzjBhfe5W9I8mK3tFJoQbEj3wB81IgQ91sh-OZbWNUcyj_GUsemdV4WrqA4pKjvT1FrTKjMTH-AMnRwV-xSrfhU1o5eSQebSGyz2axKGjEgJ9yRgxb8hK5gBQsKwukLtNaX)
###### Figure 2.9 OpenCL floating point precision test and generated output.

The only way to correctly represent a third is an infinite amount of 3s to the right of the decimal point. However, as there are only so many bits to work with in a float (32 bits in OpenCL) at one point it is no longer possible to correctly represent the exact value. Care should be taken to make sure that the value obtained for t has been correctly ‘rounded’ up or down to the nearest 32 bit representable floating point value. For intersections with spheres this is the value where the point of intersection along the ray in terms of t lies just outside of the sphere, not on, and not inside.

![](https://lh6.googleusercontent.com/xMJqBzUL01bo3bguxiBjjuJ7UOlZklC1-151kggjYrnRzq8bbjScf_bf6xtGxk1tgB_XsktxMfNl_PsYHT3xoaRUb1FF4VVUzG4NHC5zu-Ha5KqDcJrIIk6-17eqpv6hISE-vgde)
###### Figure 2.10 A light ray intersecting the sphere it was supposed to originate from.

There is a big issue with t values that put their points of intersection just on the other side of the surface that was intersected. As soon as new rays are generated from this point (for example a light ray to determine if the point is lit by any of the lights in the scene) they immediately intersect with the shape they are supposed to originate from (causing the light ray to turn into a shadow ray, even if there is a clear path to a light source from this shape).

![](https://lh4.googleusercontent.com/aUb8H85E2qYHVKjbWeuL1Tk1YPEXOChxx0ihGQ0jgdoTIBQTTQRcq-zQz9AdM7N79VeOGBgabhQmBW5ESiH_V8nc8HjQh0Os4U5B4CrYVZekFXOQoZWtKaH7uwY__mwdUSrjk6m9)
###### Figure 2.11 A sphere displaying speckles in its shadow due to precision problems.

There are a couple of ways to account for this inaccuracy. The most surefire way to move t out of the sphere as accurately as possible is to subtract the smallest value representable by a float from t while the distance between the point of intersection and the center of the sphere is smaller than the radius of the sphere. The major drawback is that this introduces a substantial amount of additional calculations and most likely branching (see ‘Fundamentals - Branching’).

A less computation intensive way is to simply ignore intersections with a t value below a certain threshold. Surely an intersection with t < 0.0001 can not be correct. But you never know. And especially when considering a simulation of atoms and subatomic particles 0.0001 is not a safe threshold at all. A better approach would be to determine this threshold for t depending on the radius of the sphere that was intersected.

![](https://lh5.googleusercontent.com/-Cp3u0j1zR-fpocOzrIZ8A_QMIZbk4dkyaY31GfVv-7s4AhAH1VY7cg7m5vQuqd7ojQqaiwxXXelqyrZ75AR-fKRicJNz9NkQ-pu_k4GDCdQzk_zDlaGZqG9Gbq0fPT4MW1srDcy)
###### Figure 2.12 Ensuring the point of intersection is outside the sphere.

I believe the best solution is to combine the two. This does mean that the point of intersection has to be computed every time we intersect an object that is closer than the previously closest object. However, this trades off favourably against having to ignore a full set computations just finished to calculate a point of intersection only to find out it should be ignored because the value for t is too small.

#### Intersecting a box

We will be looking at intersecting a ray with a box through the use of slabs. A method presented by Kay and Kajiya [5]. A box can be defined by three sets of two parallel planes, otherwise represented by a minimum extent and a maximum extent (the lower left corner of the box and the upper right corner of the box). A slab is the space between the two planes of any of these sets.

![](https://lh4.googleusercontent.com/d3uq_RzRoUoLUDjjij4Awlfk5s3DAizd5UCpmUHp73mPviN_enYmF_y9ALpI_DtguC6BFgSM_Q2ZqqUiEcZ9gcKsha5cyF7jCEPjSjEyblrDVeDTfFFV4PurZ-K9uRvw84x8vq4a)
###### Figure 2.13 A ray intersecting the horizontal slab between the two x planes at tnear and the vertical slab between the two y planes at tfar.

The following set of computations is to be executed for every set of planes. The first step is optional, you could consider leaving it out to be an optimisation. Assume the x planes are being examined. If the x component of the direction of the ray is equal to zero the ray is parallel to the planes. It follows that if the x component of the origin of the ray is not in between the two x planes (the x component of the minimum extent of the box and the x component of the maximum extent of the box) the ray does not intersect the box. The question to ask here is; how often will a floating point value be exactly zero? Is it worth having every ray carry out this check for every box while most likely only a fraction of those rays, if any, will ever make use of them.

Else, if the direction of the ray is not parallel to the set of planes currently being observed calculate the intersection distances of these sets of planes.

![](https://lh4.googleusercontent.com/pMZnuqdYv8Dc9sUn0a-0RtCn9tVEICMLvrrAPPHvakRHIgW1aM_LV56Fz9iJCVNHrIBjslrNR1NxDgWHvXXEu--SxTdKc0k_YNn5OdXt2lJG7VRWO5vuTyhtisAwKhG0DFBCjOit)
###### Figure 2.14 Before looking at any of the slabs initialise tnear and tfar.

![](https://lh4.googleusercontent.com/DIjWvofv3BSvqlEwg1IOTmKX2PV80bNUR5Bm3ZIWLtdLyqwp2TZUv5WY_OI6GsgiRb_8a9rIkNS_xJa4mDMmsAOLFrj6CgvLf9b1VDRVacELosGPLlm1oB5LKNlR0X27Qd59-r1E)
###### Figure 2.15 Calculating the intersection distances to the planes and updating tnear and tfar accordingly.

tnear should always be compared to the largest value found between t1 and t2 and tfar is to be compared to the smallest value found between t1 and t2. Check whether t1 and t2 need to be swapped and compare and assign tnear and tfar appropriately. If at any point in time the largest tnear found is greater than the smallest tfar, or the smallest tfar found is smaller than zero, the ray misses the box. Repeat the computation for the set of y and set of z planes. If in the end tnear is still smaller than tfar and tfar is still larger than zero the ray hits the box with an intersection distance equal to tnear.

![](https://lh4.googleusercontent.com/gXo7p4znfEPsT2MBlYezWQ3ae6xcFXeUAwT6EHb_3_qyCH-Qj3jjeNBZCVYFwCuVzWC-FS3HbCYNmn72DQl5pBgzfvJ-xJRxHP3fjc-j7fWHM1lO_l_5OFKC8WXK6U0bIeefH1Ir)
###### Figure 2.16 After every slab check if either tnear or tfar is no longer valid.

![](https://lh4.googleusercontent.com/bFI2zGawG22_tyOGo3x1oma6zACtLuXFOIaRPVkvzIqxetFdtSnW0SNjCHqeLRTtc4JiDM6foauBOP6-bGfw1gPhIWYOXVuQ8fikyInIqGmQbkRUraxf_Jjd4RpTOlIbGcdCrRHV)
###### Figure 2.17 Finding the normal at the point of intersection on a box.

While very convenient to work with and removing the need to specify all eight points of a box the minimum and maximum extent also impose some limitations. For example a box with only a minimum and a maximum extent will always be perfectly aligned to all three axis. But this characteristic can also be exploited when looking for the normal on a box. Set a minimum value to infinity. Calculate the direction vector from the center of the box to the point of intersection. The center of a box can easily be found by adding both extents together and multiplying the result by a half. For every value of x, y, and z calculate the absolute distance between this value of the minimum extent of the box minus the absolute of this value of the previously calculated direction. If this distance is less than the current minimum value update the minimum value to this distance and assign a normal appropriate for the value that was being worked with; either x, y, or z.

#### Bounding volumes

It was previously mentioned that bounding volumes are an incredibly powerful optimisation method as they exponentially reduce the amount of calculations needed to render increasingly more complex scenes.

A bounding volume is exactly what you would expect it to be; a volume which defines the boundaries of everything that lies within it. Notice the page you are reading right now, whether physically or digitally. It can be considered to be the bounding volume for every single character that is written on it. If you were to move this piece of paper aside or scroll up or down such that you could no longer see this particular page you no longer have to think about anything that is on it. For if you can’t see the page altogether, how could you possibly see its contents.

![](https://lh6.googleusercontent.com/FvqqREk-hNodhB7pY8RbnKQItdKXA1kaziJbsE2D_y5kD0Fmj456GBSE0cFDn2e5iRxsqm-bUZIfo8NcnjtmNgX9397SexQsDKeZaJTZn3-dWt7FNULAlqKinle3JT5O8UtK1Hj8)
###### Figure 2.18 A box and sphere bounding volume of a complex model [2] [3].

If a ray tracer can not see (read intersect) the bounding volume of a complex model it will not be able to see (intersect) any of the shapes that lie within it. By using bounding volumes a ray tracer can, at the cost of intersecting a couple thousand additional shapes, safely skip intersecting up to millions of others.

#### Intersecting a plane

Intersecting any n-vertex polygon starts by intersecting the plane it lies upon. A plane is defined by its unit vector normal and a certain distance from the origin of the coordinate system. The sign of the distance indicates on which side of the plane the origin lies. A positive sign means the origin lies in front of the normal of the plane and consecutively a negative sign means the origin lies behind the normal of the plane.

![](https://lh3.googleusercontent.com/NtzxEkvjRaLBBJyptzP816oakZkDMRYJymPMm3od7qoV_8GiClHYBo9RxNBtr1rJEFHlQa9gJCk7xxd010SYPWbW-naeDt4-65amAPPK1ytJr5HjX1ypdcyU09XBDv5FBoXLSh8Q)
###### Figure 2.19 A plane struct in OpenCL.

The first conditions to check are if the ray is parallel to the plane (they will never intersect) and whether the plane normal is pointing away from the ray (the intersection is considered illegal).

![](https://lh3.googleusercontent.com/yVGfRSzeztYaFRZsVEB6qTLgfsEQ2DA_4rKe906pC3lqrMI9xWXBX4O4ZwTSGy99M5tErU0M8lxDEhcBrypsWiG2JSmsl6jBd9Xh4p4q7jQvZC6DZ0V9oVcSLKxATQVRNCLtOuCp)
###### Figure 2.20 Testing whether a ray misses a plane.

Conveniently these conditions can both be derived from the dot product between the normal of the plane and the direction of the ray. Whereas if the result of the dot product equals zero the ray has a direction parallel to the plane and if the result is larger than zero the plane normal is pointing away from the direction of the ray resulting in it being culled. In both cases we can leave the computation early. If both these tests were passed calculate a second dot product and finally the ratio between them.

![](https://lh3.googleusercontent.com/lsJ6oBPXFNZj4m9sjj1pT3bJpf7nL1kitb1dKIaEiheoXoI7hd8-oAqwPvaxW5pTw61031NoHpKcutsPBCMSdSqv5plrIEc8MlzOIPrftK0kxyvhUhrM22QB_ZftH1Ev2-yXj60_)
###### Figure 2.21 Testing whether a ray intersects a plane behind its origin.

Planes stretch towards infinity and any ray that is not parallel to it will eventually intersect it. The question is whether it does so in front of the ray or behind it. An intersection behind the origin of a ray (where t is negative) is considered illegal as the point is not technically on the ray, it is a point on the line along which the ray travels. Intersections where t equals zero are considered illegal for reasons explained in ‘Fundamentals - Precision problems’.

![](https://lh5.googleusercontent.com/dCZ8RZsdkzc1f3ZOoP3UkFdc3x5bjNFVcpCyd18MtF6Dto7yG6PevQwWQ7Oss2vJ-YDukx88mYhq9Mcn-piJG4-2U0QX21Fno9L8r7Vx6CpdyajYcUbVYH1eU7VvLFsEIevNVrfl)
###### Figure 2.22 Testing whether a ray misses a plane and flipping the plane normal if it is pointing away from the ray.

Where it is desired to render planes regardless of orientation it is possible to occasionally flip the normal of a plane such that it always faces the ray. If the dot product between the normal of the plane and the direction of the ray (Figure 2.20) is larger than 0, instead of leaving the computation early, flip the sign of the plane normal and restart the computation.

#### Branching

OpenCL works with an SIMT (single instruction, multiple threads) execution model. This means that multiple threads have the same program counter and will execute the same instructions at the same time. One huge performance killer in parallel computing is branching. As soon as even one thread in the group has to take a different path from the rest it has to be taken out and put into its own execution path.

![](https://lh4.googleusercontent.com/g2U5fwtyhP2dbcfxRAmpYfS4Rl0p-LsFlMwkiMa4qTDRGdyjrkqtHgiiSz7bE2D_-tJ3S8i8qlCjjmUggGmlhdoGogYllZ3dDYeIWbAfd4BXVy4p1Tj4KZ3wUABUXEl5OAmitqHT)
###### Figure 2.23 Branching can be avoided.

There are several tricks through which branching can be avoided. Most of these include clever use of conditional statements in line with assignments and calculations. It is usually even worth considering to do more computations if it means no branching will occur. Figure 2.23 states that in the last assignment there is no choice but to branch as threads where the condition is true can not simply assign the function call to the direction variable. This function has to be executed. But what if every thread in the pool executes this function regardless of the condition.

![](https://lh5.googleusercontent.com/3PSKEpi43JjSk0xC1RlE49MnEzczrC06uSj-1Re2jYecQsHb2kMOuigQtngbuQheF5oCWrJnWcOh2S5ZwaYmXkXEzsjjoOgXD-vapKySZTFexUbv_Byt_ad5sf31zuctKtWeu5S3)
###### Figure 2.24 Branching has been completely avoided.

Every thread now always executes the exact same thing. As long as there was at least one thread that was going to branch this is a performance gain. Executing the code yourself alongside another thread in comparison to having to wait for that threat is not going to cost or win you any time. The overhead that was avoided by removing branching however is a significant gain. Do note though, if none of the threads were going to branch every single one of them is now performing unnecessary computations. While often there is a clear approach to be taken when having to deal with branching this is not always the case. Evaluate and treat the situation on a case by case basis.

![](https://lh5.googleusercontent.com/Fs3TXHWV7NkdRRA-8DvW1xLVaatv4HMxK9F10m2CvCNG-UnA_fzOH9EtoFu1uamsSlHahUesOR8VXlPs8D6hJadc-0FmUCbHXd39Q8OHNlugVdno-jrNyWx-r6pEfn8cjdVVHxKs)
###### Figure 2.25 A barrier makes whichever branch finished first wait for the other.

When it is absolutely impossible to work around branching, or where it would otherwise impede the architecture of your implementation barriers offer a perfect solution. A carefully placed barrier will make sure that any branch that reaches the barrier waits for those that have not done so yet. Consider it a catch all mechanism to get all threads back together again. Though be careful, as exactly every thread in a thread pool or exactly none of them has to reach a barrier. As soon as one thread in a pool has reached a barrier all other threads from that pool much reach it or the kernel ends up in deadlock. Make sure to never branch around barriers. Remember the computations we looked at earlier to ray trace a plane. There were multiple conditions to look at whether the calculations could be halted early as the ray missed the plane. By placing a barrier right after this function it is guaranteed that every thread in the pool leaves the function together. We can benefit from leaving the function early in the case where every thread does so (and thus all of them reach the barrier sooner) and we don’t have to worry about threads leaving the function early getting ahead of the other threads resulting in even more branching along the way.

Shading
---

Having found an intersection it is still required to compute the colour that is supposed to be displayed at this point. While it will produce some quick results, simply taking the colour from the material of the object that was intersected does not look very realistic.

![](https://lh4.googleusercontent.com/UR3BOHL1jMGradNQmj8sbIUUBcdkxKSHeyFwGHeO-fal2R3lfWF2zRPs0HJIygm5PgWE5foKGVFYmN9KmqfEkTSEeL3cDhC7sA_5NljDlOAcx9jdYnNqEXLQC730jd8fk3AYVADx)
###### Figure 3.1 A poorly shaded sphere without any light rays or reflection.

#### Light rays

The first thing to do when the closest point of intersection has been found is to shoot a light ray from the point in the direction of every light in the scene. This ray is easily constructed. The origin of the ray is the point of intersection and direction of the ray is the normalised value of the location of the light minus the point of intersection.

![](https://lh4.googleusercontent.com/wFmtFQd1OxtF3BF6UU2gSBf1Ils_CO372640sEcowb2322j919FSd9lM94e-gAXYKhRxaTQ2Lbro74-yd0uerbpw1PxypUenWEEWU9wRxZ86O38VsVm-IGWjHIPgW3mQ_SGwkqnB)
###### Figure 3.2 Two light rays, one which was successful in reaching the light source and one which turned into a shadow ray due to intersecting an object in the scene.

Once a light ray has been validated by reaching its light source the colour and intensity of said light can be applied to the material that was found at the point of intersection. But this is going to give incredibly bland and unrealistic lighting across the scene. We are looking for a value, preferably between zero and one, that is closest to one at the point on the object that is closest to the light source.

![](https://lh5.googleusercontent.com/C_MWJHcbvzymtFmKD9OR5042NaFUiGo8qh9i02hf78G5JIqu-gAmVgDDLhSbkH1awlEXYTQ3Zg87ImdqvJbK9V8V2WaTX0TjRUfst95nLiBzsDsr1NWXYPu82rJRoNdOVEr22je3)
###### Figure 3.3 An easy way of applying a gradient to diffuse lighting.

Points where their normal is pointing in a very similar direction as the direction of the light ray will be well lit. Whereas points with normals that are pointing in a rather perpendicular fashion to the direction of the light ray will have the light slightly brush over their surface. There is one such value that bounces in between one and zero as two vectors get close to and further away from each other respectively. Take the dot product between the normal at the point of intersection and the direction of the light ray. Multiply this with the colour of the light source and ultimately multiply all of this with the colour of the material at the point of intersection.

![](https://lh5.googleusercontent.com/5ehU12GKd_8NZEMX2GN-0UGYftvj6PI1ZggLLfpi3KUzupMCrB_8OyxaB3X-u2aHksr8bBJtfxa4vHgDiD7Ktl4z0IkBDpWX0fyWG_lcpPz3nQJzD2m_7myw-cc_mExU1MKF5gJj)
###### Figure 3.4 A softly shaded sphere.

#### Shadow rays

A light ray which never reaches its light source is considered to be a shadow ray. Shadow rays do not light the points from which they originated. A point of which all of its light rays turned into shadow rays will remain in the dark. Instead of using the colour of the material at the point of intersection a black can be used instead. One can also experiment with a mixture of the two, where for example 90% black is added to the colour of the material at the point of intersection.

#### Reflection

When a surface has a certain reflectivity, in addition to spawning light rays at the point of intersection, we should also create a reflection ray. There are multiple types of reflection. Perfect reflection, otherwise known as mirror reflection, is the simplest type of reflection to implement. The law of reflection states that for perfect reflection the angle of incidence is equal to the angle of reflection.

![](https://lh3.googleusercontent.com/CtMVlsHCK-wJbiKB0L1lo7Tar2Otf7XEIv3oOkXhyiVJO1km32Jdm-Ae5DYFbyyWSW9KRj6hNGxl62S4RgIwGoaob-2khMC5ghEAWPIM0LkFZu8UVob_tgYgqb8Rd8Wyu0kL0-yj)
###### Figure 3.5 Given a certain incident and normal a reflection ray is computed.

![](https://lh6.googleusercontent.com/AQumo2pel6ncQJ223r-63H_ejV_9Bwfx_LXMxEsOjqibPh2MLgw-kV9hz4HrxXQmIR--OO-erhMh_ziVOZlZhx0fNzT3fjouTKgwZgnHKMv46j2KvKAR-ITVxt0X_HGjcdGqteYc)
###### Figure 3.6 A softly shaded sphere with a certain reflectivity.

#### Phong

Bui Tuong Phong developed the idea of being able to compute materials via the sum of its weighted diffuse and specular attributes.

![](https://lh6.googleusercontent.com/JOnt1am0Rp_AQGDIfjdatflCH_vr5fi5_YWe9qFXsF7RNwxdAojBAjtXRFFi5DVyQNZuK5OtQErwXCPMd78qVocttTk4nDhFEYJM4LVm9lO0dvfNBuUqKJHBfYbIbkKw_9E7nYla)
###### Figure 3.7 Calculating the specular component of Phong’s model.

Calculate the reflection ray using the reverse direction of the light ray as the incident and the normal of the point of intersection. Project this ray backing onto the reverse direction of the original ray and raise it to the power of your specular constant. The higher this constant the more concentrated the specular reflection.

![](https://lh3.googleusercontent.com/qGtnQsu-Aex0Y9sr4mma8yYvfEdjnGXkxETrLAuZ6sk2Or5ws_WgL_UvXTMgfiDrNy3lop-sEFik4C8XOGZ7BQD1n77GucCDnt6M2nolwguK6WaxTB80zRELNRbXiYJFCvl7Yc57)
###### Figure 3.8 A sphere with a glossy reflection on top.

Evaluation
---

I have ran into many issues with programming languages, platforms, you name it. I started off this project as a ray tracer that was to be written in C++ and OpenGL. It was the language I used at my previous university and I thought it would be nice to go back to it. I was wrong. I have been spoiled by Java’s consistency and semantics and ended up pulling my hair out trying to get anything done in C++. Furthermore after making the transition to Java and OpenCL after realising that OpenGL really doesn’t have much in common with ray tracing I struggled quite a bit with getting the OpenCL context to initialise. This eventually led to demoralisation and it took a while for me to pick the project back up. To this day I still can not run the kernel on my GPU.

It is no secret that this project ran out of time. I haven’t nearly been able to go into as much depth of most topics as I would have liked to and this comes down to planning. It is a recurring issue and I will continue to work on it.

But mostly I have had my mind blown multiple times, that not only does ray tracing include complex computations, but even the most simplest of calculations can greatly increase the aesthetics of the render.

Conclusions
---

It is clear that while ray tracing is an incredibly elegant algorithm on the outside the depth of the underlying mathematics is endless. I have worked on this project with great pleasure and regret that there are still so many sections to explore for which I did not have the time. I will be taking this ray tracer with me into the future as I expand upon it with refraction, specular reflection, diffuse reflection, dielectric materials, directional lights, soft shadows, and more such that one day it will be the great piece of software it should have been today.

![](https://lh6.googleusercontent.com/reaz1lnsUl-IyrMu_pQHzaCYsTIx8UtXrTpWSlxK7WSVcBuFP-CQbMc8PupQwkA2lT1TaE69jBQdYSll09tX3LQ2BGmZB5LKMNsgzQWjNlLirZvCRns2cBG1hMuHOeuxCz8G7sd_)
###### Figure 4.1 A render of spheres with diffuse and specular lighting and reflection.

Bibliography and References
---

1. Glassner, Andrew S. (1989). An Introduction To Ray Tracing. London, Academic Press Limited.

2. Chanj. (2011). Bounding box [ONLINE]. Available at: http://mathforum.org/mathimages/index.php/Image:Bounding_box.png [Accessed January 2016].

3. Chanj. (2011). Tighter bounding sphere [ONLINE]. Available at: http://mathforum.org/mathimages/index.php/Image:Tighter_bounding_sphere.png [Accessed January 2016].

4. Duff, T. (July 1984). Numerical Methods for Computer Graphics. Siggraph Course Notes, Vol. 15.

5. Kay, T.L. and Kajiya, J.T. (1986). Ray Tracing Complex Scenes. Siggraph '86 proceedings of the 13th annual conference on computer graphics and interactive techniques, pages 269 - 278.

6. Scratchapixel. (2009 - 2015). Computing the pixel coordinates of a 3D point [ONLINE]. Available at: http://www.scratchapixel.com/lessons/3d-basic-rendering/computing-pixel-coordinates-of-3d-point [Accessed February 2016].

7. Scratchapixel. (2009 - 2015). Rasterization: a practical implementation [ONLINE]. Available at: http://www.scratchapixel.com/lessons/3d-basic-rendering/rasterization-practical-implementation [Accessed March 2016].

8. Scratchapixel. (2009 - 2015). Ray tracing: Generating camera rays [ONLINE]. Available at: http://www.scratchapixel.com/lessons/3d-basic-rendering/ray-tracing-generating-camera-rays [Accessed March 2016].

9. Scratchapixel. (2009 - 2015). Introduction to shading [ONLINE]. Available at: http://www.scratchapixel.com/lessons/3d-basic-rendering/introduction-to-shading [Accessed April 2016].

10. Scratchapixel. (2009 - 2015). The Phong model, introduction to the concepts of shader, reflection models, and BRDF [ONLINE]. Available at: http://www.scratchapixel.com/lessons/3d-basic-rendering/phong-shader-BRDF [Accessed April 2016].

11. Scratchapixel. (2009 - 2015). Global illumination and path tracing [ONLINE]. Available at: http://www.scratchapixel.com/lessons/3d-basic-rendering/global-illumination-path-tracing [Accessed April 2016].

12. The Khronos Group Inc. (2007 - 2011). OpenCL documentation 1.2 [ONLINE]. Available at: https://www.khronos.org/registry/cl/sdk/1.2/docs/man/ [Accessed 2016].

User guide
---

Use of the program has been kept as simple as possible. Keys can be held down to navigate through the scene both horizontally as well as vertically. In addition, the viewing angle can be adjust freely by turning the camera around.

##### Moving horizontally

    Forwards                'W' key or '↑' key

    Backwards               'S' key or '↓' key

    Strafe left             'A' key or '←' key

    Strafe right            'D' key or '→' key

##### Moving vertically

    Up                      'Spacebar' key

    Down                    'Shift' key

##### Turning the camera

    Holding down the right mouse button and moving the
    mouse allows for looking up, down, left, and right.
