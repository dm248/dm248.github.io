# Where's the Sat @ Hack-A-Sat

##### Jun 7, 2020

Two weeks ago, the USAF and DDS put up a [Hack-A-Sat](https://www.hackasat.com/) space hacking challenge.
I only had time to check out a handful of the problems, one of which was Where's the Sat that asked people
to identify a satellite by location and time and then predict where it would be at another time. It turns out
that with the
right tools, such as [Skyfield](https://rhodesmill.org/skyfield/), the problem is straightforward 
(check, e.g., [here](https://medium.com/@pdelteil/wheres-the-sat-hack-a-sat-writeup-9a523634963b)). 
But the solution is not very satisfying unless one understands what goes on inside the black box... 
so if you are curious, read on.

### Remember Kepler?

To first approximation, Earth is spherically symmetric and satellites are pointlike, in which case orbits are
ellipses with the center of Earth at one of the foci 
(recall [Keplerian orbits](https://en.wikipedia.org/wiki/Kepler_orbit)). E.g., if you picture the "standard" 
ellipse in the *x-y* plane with 
semiaxes *a* and *b* along the *x* and *y* axes, and shift Earth to the origin, then the center of the ellipse
will be at (-*a* *e*, 0), while the point of closest approach (perigee) at *P* = (*a* (1-*e*), 0),
where *e* is the eccentricity.

Sats in the challenge were given as a list of NORAD two-line elements 
([TLEs](https://en.wikipedia.org/wiki/Two-line_element_set)). 
These contain the parameters of the orbit and where on the orbit the sat was at the given time, 
specifically, the 

* epoch = year and solar day, including fractional days, at which the TLE data apply   
  In case you missed the memo, TLE years start with [day 1.000...](https://www.celestrak.com/columns/v04n03/) 
(not 0.000...).
* inverse period 1/*T* (also gives the major semiaxis *a* via [Kepler's 3rd law](https://en.wikipedia.org/wiki/Kepler%27s_laws_of_planetary_motion#Third_law_of_Kepler))
* eccentricity *e* (together with *a* gives the minor semiaxis *b*)
* argument of perigee = angle by which the standard ellipse needs to be rotated counter-clockwise in the *x-y* 
plane (fixes the orientation of the orbit in the orbital plane)
* inclination = angle by which the rotated ellipse needs to be tilted about the *x* axis (i.e., the orbital plane 
tilted out of the *x-y* plane)
* right ascension of the ascending node = angle by which the tilted plane is rotated about the *z* axis (this 
final rotation fully sets the plane of the orbit)
* mean anomaly = position on the orbit at epoch, given as 2π times the fraction of the ellipse's area 
swept - since perigee - by the line connecting the sat and the origin

The list in the challenge had sats with ballpark 1.5-hour orbital periods,
zipping around at roughly 6700 km from the center of Earth -
such as the International Space Station (ISS).

With the help of some geometry (if you are rusty, check [Wikipedia](https://en.wikipedia.org/wiki/Kepler_orbit)),
you can reconstruct the ellipse and the position on it
corresponding to the TLE. 
The result is not terrible but still misses by a lot; e.g., for the ISS it was off
by 30 km already *at* epoch and by more than 150 km within ~5 orbits after epoch.
Orbits are not ellipses because Earth's gravitational field is not perfectly spherical plus
there is atmospheric drag, so Kepler's solution no longer applies
(other bodies such as the Moon and the Sun also exert forces but for orbits near Earth
those are less relevant).
Short portions of orbits can always be considered elliptical but parameters change along the path.
Moreover, one cannot use TLE data directly even near epoch because TLEs contain averaged quantities,
not instantaneous values 
(thanks to Dr. Kelso at [CelesTrack](https://celestrak.com/) for pointing that out to me).

### Imperfect Earth

The most important correction is that Earth's gravitational potential *U* (*r*,ϑ) 
varies with latitude. People usually [expand](https://en.wikipedia.org/wiki/Geopotential_model)
the potential over Legendre polynomials in sin ϑ,
and keep the first couple terms, say, up to the fourth-order term - the coefficients of which (*J2*, *J3*, *J4*) 
are determined by measurements (*J1* is zero by construction). Gradients of the terms beyond zeroth order 
then give additional force contributions that one can plug into 
Newton's ***a*** = ***F*** /*m* and find sat trajectories
via integrating the differential equations numerically
(about freshman physics at most places, though a 
[high-order](https://en.wikipedia.org/wiki/Runge%E2%80%93Kutta_methods)
integrator does not hurt).
However, this approach still requires initial conditions, 
so one cannot sidestep extracting instantaneous positions and velocities from TLEs... somehow.

TLEs are designed to work with a certain class of orbital propagation models. 
A popular one of these, applicable to near-Earth trajectories, is
Simplified General Perturbations version 4 ([SGP4](https://en.wikipedia.org/wiki/Simplified_perturbations_models)).
The model stems from *approximate* integration of Newton's equations 
in convenient coordinates for Keplerian motion, which yields tractable math
for small *J2*, *J3*, *J4* corrections to gravity and a weak drag force.
The simplified math really mattered in the old days when computers were slow. 
The algorithm with the resulting equations is publicly 
[available](https://www.celestrak.com/NORAD/documentation/spacetrk.pdf)
(in [Python](https://github.com/brandon-rhodes/python-sgp4), too, by now)
but for the actual physics papers 

* *Brouwer*, Astronomical Journal 64, p.378 (1959)
* *Lane*, AIAA Paper 65-35 (1965)

behind it you will likely need access to a research library. 
TLEs contain averaged orbit parameters that are convenient
input quantities for SGP codes, and the codes then give
position and velocity at any time, both before and after epoch (with errors that grow with the 
time difference relative to epoch).

It seems like we are done: we can either use SGP4 for everything,
or take initial conditions at epoch from SGP4 and then solve Newton's equation numerically.
However, if we try the latter for the ISS example above, 
we get a discrepancy of about 4 km per day from epoch compared to SGP4 predictions.
This is worrisome because SGP4 is known to work very well in practice, and 
the task in Where's the Sat is to predict positions at about 23 days prior to 
epoch, at which point our real physics calculation is off by nearly 100 km. 


### Drag

Earth's atmosphere dilutes with altitude but even a small air resistance can matter for sat motion.
SGP4 uses [quadratic](https://en.wikipedia.org/wiki/Drag_(physics)#Drag_at_high_velocity)
***F*** = -const ρ|v|***v***  drag, written as the acceleration ***a*** = -(B* /R) (ρ/ρ0) |v| ***v***,
where B* is specified in the TLE, while *R* is the radius of Earth.
The density ratio is of a power-law form `[(q0 - s)/(r - s)]**tau` (see Lane's paper), 
where q0, s, tau are fit parameters. Specific values can be obtained by inspecting the code:

* q0 = R + 120 km, 
* s = R + 78 km
* tau = 4

(for r < R + 158 km, SGP4 uses a different set).
With air resistance included in Newton's equations,
the deviation from SGP4 reduces by half for the ISS (to 2 km/day), which is good but not good enough.

Is 2 km/day simply the inherent accuracy of SGP4, or does the model have some other physics? 
Actually, no and no - the reason for the discrepancy is more subtle. 


### Initial conditions

SGP4 is a good approximation for an extended time around epoch. 
But its answers are only approximate, and 
therein lies the problem when we want to use SGP4 positions and velocities as initial conditions for 
Newton's equations. 
In fact, with just about one part per million adjustments to initial conditions 
from SGP4 (couple tens of meters in position and few *milli*meters per second in velocity),
one *can* obtain Newtonian trajectories that stay close to the SGP4 solutions for a long time.
 
A reasonably intelligent way to find these adjustments is to systematically alter each component 
of the initial position and velocity by a small amount (six degrees of freedom), 
and then find the linear combination of those changes that brings the position predicted by Newton's equations 
in agreement with SGP4 at two points in time, say, about one period before and one period after epoch. 
An even better way would be to [minimize](https://en.wikipedia.org/wiki/Least_squares) 
the sum of deviations at multiple points along one orbit
(I have not explored that).
Either way involves some straightforward linear algebra. 
From the adjusted initial conditions,
the Newtonian trajectory agrees with SGP4 to about 100 meters/10 days from epoch
for the ISS when both nonspherical gravity corrections and drag are included.

This translates to roughly 200-meter accuracy at twenty-some days (~300 orbits) before epoch, 
which would have been 
plenty for the Where's the Sat challenge
(as far as I can tell, the challenge was coded to tolerate up to 1% error, i.e.,
couple tens of kilometers).

### TL;DR

For accurate satellite orbit predictions you must go beyond Keplerian orbits and take into account the 
nonspherical gravitational field of Earth and also atmospheric drag. 
Straightforward physics, in principle, but getting precise-enough initial conditions for Newton's equations
from TLEs is tricky. 
Thus, solving Where's the Sat in the available timeframe was basically impossible 
without applying SGP4 or a similar propagation model.

---

[[back]](/)
