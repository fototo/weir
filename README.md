# WEIR-A System for Making Generative Systems


## About

This library is specifically written to be useful for a broad range of ways in
which I create art using various generative algorithms. It is the next
iteration of [snek](https://github.com/inconvergent/snek). I made a new version
because I wanted to make some significant changes.

Main components:

1. 2d and 3d vectors with corresponding operations:
   ```lisp
   ; create a 2d vector
   (vec:vec 1d0 3d0)

   ; create a 3d vector
   (vec:3vec 1d0 2d0 3d0)
   ```
   `vec` supports common vector operations such as `add`, `mult`, `div`, `sub`,
   as well as cross products `cross`, dot products, `dot`.

   *Note: Functions that operate on 3d vectors are prefixed with `3`.*

   Furthermore there are corresponding functions for scalars, lists of vectors,
   and broadcasting. They are indicated by prefix `l` and `s`:
   ```lisp
   ; add scalar s to a, return result
   (vec:sadd va s)

   ; add vectors in two lists, returns list of new vectors
   (vec:ladd aa bb)

   ; add b to elements of aa, return list of new vectors
   (vec:ladd* aa vb)
   ```

   Most of these functions also have a corresponding function postfixed with `!`,
   which performs the same operation, but assigns the result to the first
   argument. This is faster since there is no need to create a new `vec` struct.

   *Note: This can cause strange behaviour since you can inadvertently change the
   internal state of a struct. Use with caution.*

   Example:
   ```lisp
   ; add b to a, store result in a; also returns a
   (vec:add! va vb)
   ```
   There are also some common geometric operations, such as line interpolation,
   line-line intersection, angles, and line-plane intersection.
   ```lisp
   ; find the position s between a and b. s should be between 0 and 1
   (vec:on-line s va vb)
   ; find intersection of two lines, returns
   a bool, and interpolation value along line 1, line 2
   (vec:segx line1 line2)
   ```
   See the code in the package for more details.

2. A simple graph data structure, `weir`, for working with vertices and edges.
   The structure is combined with a programming pattern for applying changes to
   the structure. The pattern relies on `alterations`, see below. You can also
   manipulate the structure directly.

   Below is a small example:
   ```lisp
   (let ((wer (weir:make))) ; use :dim 3 for 3d
     ; add three edges
     (loop repeat 3
           do (weir:add-edge! wer
                (weir:add-vert! wer (rnd:in-circ 200d0))
                (weir:add-vert! wer (rnd:in-circ 200d0
                                :xy (vec:rep 500d0)))))
     ; iterate verts
     (weir:itr-verts (wer v) (print (weir:get-vert wer v)))

     ; move a vert relativ to current position:
     (weir:move-vert! wer 0 (vec:vec 1d0 2d0))
     ; move a vert to an absolute position
     (weir:move-vert! wer 1 (vec:vec 1d0 2d0) :rel nil)

     ; iterate edges
     (weir:itr-edges (wer vv) (print (weir:get-verts wer vv)))

     ; edges are represented as lists of verts, and they are always
     ; sorted with the smallest vert index first, so both
     (weir:edge-exists wer '(0 1))
     ; and
     (weir:edge-exists wer '(1 0))
     ; returns t

     ; get edges incident to vert 0
     (weir:get-incident-edges wer 0))
   ```
   See the `examples` folder for more.

3. A series of other useful data structures and tools. E.g. a package for
   generating different kinds of random numbers (`rnd`), as well as tools for
   handling colors (`pigment`), splines (`bzspl`), and various vector and path
   functionality. Eg. `math`, `lin-path` and `simplify-path`.
   ```lisp
   (let* ((points (rnd:nin-circ 5 400d0))
          (bz (bzspl:make points))
          (lp (lin-path:make points)))

     ; sample a point on the spline
     (bzspl:pos bz (rnd:rnd))
     ; sample a point on path
     (lin-path:pos lp (rnd:rnd))

     ; represent the spline with a limited number of points
     (bzspl:adaptive-pos bz :lim 1d0))

     ; return n numbers between a and b inclusive
     (math:linspace n a b :end t)

     ; return n random numbers between a and b
     (rnd:rndspace n a b)

     ; pick a random point between points va and vb
     (rnd:on-line va vb)
   ```

4. Orthogonal projection `ortho`:
   ```lisp
   (let ((proj (ortho:make :s 1d0
                           :xy (vec:rep 500d0)
                           :cam (vec:3vec 1000d0 1000d0 0d0)
                           :look (vec:3zero))))
     (multiple-value-bind (v d)
       (ortho:project proj (rnd:3in-sphere :rad 300d0))
       ; point in 2d
       (print v)
       ; distance from 3d point to camera plane
       (print d))

     ; update cam position and look at something else
     (ortho:update proj :cam (vec:3vec 3d0 4d0 1d0)
                        :look (vec:3rep 79d0)))
   ```

5. A tool for drawing `pngs` called `sandpaint`. This package uses random
   sampling to draw. This creates a fairly distinct and gritty look in many
   cases.

6. A tool for drawing `svg` files: `draw-svg`. Mainly files that are good for
   plotting.

7. Some simple raytracing functionality for triangle meshes `mesh` and point
   clouds `point-cloud`. This is immature and slow.


## Weir Graphs and Alterations

An `alteration` is a change that will be applied to the structure at the end of
a given context. In many ways this is similar to map/reduce.

Here is and example of manipulating a `weir` instance called `wer` using
`alterations`. Alteration constructors are postfixed with `?`.

```lisp
; context start
(let (wer (weir:make))

  ; add some data to wer here ...

  (weir:with (wer %)
    ; iterate vertices
    (weir:itr-verts (wer v)
      ; move alteration
      ; (% ...) is used to accumulate alterations
      ; they are not performed until the end of (weir:with ...)
      (% (weir:move-vert? v (rnd:in-circ 1d0)))
      ; w will be an arbitrary
      ; vertex in wer
      (weir:with-rnd-vert (wer w)
        ; join v and w if they are closer than d
        (if (< (weir:edge-length wer v w) d)
          ; join vertices alteration
          (% (weir:add-edge? v w)))))
    ; context end. alterations are applied here.
    )
```
You can define your own arbitrary alterations (functions). There is an example
of this in `examples/custom-alt.lisp`.


## Use

I use `weir` for most of the work that I post online
(https://inconvergent.net/, https://img.inconvergent.net/,
https://twitter.com/inconvergent). Both for raster images, as well as vector
images for plotter drawings.

Here are some plotted examples:

  - https://inconvergent.net/2017/spline-script-plots/
  - https://inconvergent.net/mechanical-plotter-drawings/
  - https://inconvergent.net/mechanical-plotter-drawings/3/
  - https://inconvergent.net/mechanical-plotter-drawings/5/


## Writing

I have written about things related to this code (when it was called `snek`) at:

  - https://inconvergent.net/2017/snek-is-not-an-acronym/
  - https://inconvergent.net/2017/a-method-for-mistakes/
  - https://inconvergent.net/2017/arbitrary-alterations/
  - https://inconvergent.net/2017/grains-of-sand/
  - https://inconvergent.net/2017/a-propensity-for-mistakes/


## On Use and Contributions

This code is written for my personal use, and parts of it is rather
experimental. Also, it is likely to change at my whim. For this reason I don't
recommend depending on this library for anything.

I release it publicly in case people find it useful or interesting. It is not,
however, intended as a collaboration/Open Source project. As such I am unlikely
to accept PRs, reply to issues, or take requests.


## Installation and Dependencies

This code requires Quicklisp to install dependencies (which are listed in
`weir.asd`). To install and load Weir, do:
```lisp
(ql:quickload :weir)
```
If this does not work, Weir may not be in a place Quicklisp or ASDF can see
them. To fix this, either
```lisp
(load "weir.asd")
```
or, for a long term solution, push the directory in which Weir sits to the
variable `quicklisp:*local-project-directories*`:
```lisp
; in your .sbclrc, for example:
#+quicklisp
(push "/path/to/dir/containing/weir" ql:*local-project-directories*)
```

The `fn` package (for generating file names) depends on the `fn` command from
https://github.com/inconvergent/fn, but this is not necessary to use any of the
other packages.

The code has only been tested in `Ubuntu 18.04 LTS` with `SBCL 2.0.1`.


## Tests

Run:
```lisp
(asdf:test-system :weir)
```


## In Case of QL Version Issues

See http://blog.quicklisp.org/2011/08/going-back-in-dist-time.html

Summary:
```lisp
(use-package :ql-dist)
; see versions
(available-versions (dist "quicklisp"))
; select a dist version
(install-dist
  "http://beta.quicklisp.org/dist/quicklisp/2019-03-07/distinfo.txt"
  :replace t)
```


## Thanks

I would like to thank:

  - https://twitter.com/RainerJoswig
  - https://twitter.com/jackrusher
  - https://twitter.com/paulg
  - https://twitter.com/porglezomp

Who have provided me with useful hints and code feedback.

The ASDF config and test setup was kindly suggested and implemented by Robert
Smith (https://twitter.com/stylewarning). The remaining weirdness in the test
system is my fault. Hope to fix it properly later.

Also, many thanks to https://twitter.com/xach for making Quicklisp.

