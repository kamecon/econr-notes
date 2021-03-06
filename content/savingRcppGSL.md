Solving a simple 2-period Consumption problem with RcppGSL
========================================================

In this example we demonstrate how to solve the following problem using the RcppGSL package. GSL is the [GNU scientific library](https://www.gnu.org/software/gsl/) and has many useful functions written in C++. RcppGSL is the Rcpp interface for R. Please look at the separate chapter "how to install RcppGSL" as that needs a trick or two.

## problem

$$ \begin{array}{ccc} max_x &\log(a - x) + \beta log(x) \\
                           &x \in [0.1,11],\\
                           &a \in {2,3,\dots,10}
                     \end{array}$$

We have a two-period problem of a consumer who has a cake of size $ a $ and needs to decide how much to consume today and how much to leave for tomorrow. The choice of how much to save, $ x $, is a continuous variable; The size of the cake today is continuous as well, but we measure it only a set of discrete points. If we are interested at the value of having a cake size of, say, $ a = 2.7 $, we can use an interpolation technique to compute this value.

This is very simple problem. To add at least some realism, we will linearly interpolate the "future value" $ log(x) $. This should mimic a more realistic situation when our model does not admit a closed form of this object.

### Solution

The good thing is that there is an analytic solution to this. Taking the derivative w.r.t. x one obtains

$$ \begin{array}{cccc} \frac{1}{a-x}        &=& \beta \frac{1}{x} \\
                     \beta (a-x)           &=& x \\
                     \beta a               &=& x (1+\beta) \\
                     a\frac{\beta}{1+\beta} &=& x 
                     \end{array} $$
                     
which implies that savings $x$ are a constant fraction of current assets.


```r
library(RcppGSL)
```

```
## Loading required package: Rcpp
```

```r
library(inline)
```

```
## Attaching package: 'inline'
```

```
## The following object is masked from 'package:Rcpp':
## 
## registerPlugin
```

```r

# define asset and savings grid
a <- seq(2, 10, le = 9)
x <- seq(0.1, 11, le = 50)

Beta <- 0.95
verbose <- FALSE  # set FALSE to supress printing the trace of the optimizer
```


### Rcpp Inline

We will use the inline package to write C source code inline. Then we compile the source in place to a C function, and we call it from R. See chapter "intro to Rcpp".

### GSL

It is useful to get familiar with the funcionality of the GSL library first. Please have a look around the manual of GSL relating to [one-dimensional minization](https://www.gnu.org/software/gsl/manual/html_node/One-dimensional-Minimization.html#One-dimensional-Minimization). In particular the setting up of the various objects can be a bit confusing initially. We are basically presenting a modified version of the example shown on that page.

### RcppGSL

First, define a string "inc" to be our personal header file for the C function:


```r
inc <- '
#include <gsl/gsl_min.h>
#include <gsl/gsl_interp.h>

// this struct will hold parameters we need in the objective
// those contain also the approximation object "myint" and 
// corresponding accelerator acc
struct my_f_params { double a; double beta; const gsl_interp *myint; gsl_interp_accel *acc; NumericVector xx;NumericVector y;};

// note that we want to MAXIMIZE value, but the algorithm
// minimizes a function. we multiply return values with -1 to fix that.

double my_obj( double x, void *p){
  struct my_f_params *params = (struct my_f_params *)p;
	double a                   = (params->a);
	double beta                = (params->beta);
	const gsl_interp *myint    = (params->myint);
	gsl_interp_accel *acc      = (params->acc);
	NumericVector xx           = (params->xx);
	NumericVector y            = (params->y);
        
	double res, out, ev;

	// obtain interpolated value

	ev = gsl_interp_eval(myint, xx.begin(), y.begin(), x,acc);

	res = a - x;	//current resources

  // if consumption negative, very large penalty value
	if (res<0) {
		out =  1e9;
	} else {
		out = - (log(res) + beta*ev);
	}
	return out;
}
'
```



### testing the objective function

It's very important to always check the objective function. here's a code snippet that will print the value of the objective at different asset and savings states:


```r
src <- '
NumericVector a(a_);	// a has the same memory address as a_
NumericVector x(x_);
double beta  = as<double>(beta_);
NumericVector ay = log(x);  // future values

int m = x.length();
int n = a.length();

// R is the matrix of return values
NumericMatrix R(n,m);

//initiate gsl interpolation object
gsl_interp_accel *accP = gsl_interp_accel_alloc() ;
gsl_interp *linP = gsl_interp_alloc( gsl_interp_linear , m );
gsl_interp_init( linP, x.begin(), ay.begin(), m );

gsl_function F;

//point to our objective function
F.function = &my_obj;

// --------------------------------
// loop over rows of R: values of a
// --------------------------------

for (int i=0;i < a.length(); i++){

  // define the struct on that state:
  // notice that the value of the state a(i) changes
	struct my_f_params params = {a(i),beta,linP,accP,x,ay};
	F.params   = &params;

  // uncomment those lines to print intermediate values
	//Rcout << "asset state :" << i << std::endl;
	//Rcout << "asset value :" << a(i) << std::endl;

  // --------------------------------
  // loop over cols of R: values of x
  // --------------------------------

	for (int j=0; j < m; j++) {
		//Rcout << "saving value :" << x(j) << std::endl;
		//Rcout << GSL_FN_EVAL(&F,x(j)) << std::endl;
    R(i,j) = GSL_FN_EVAL(&F,x(j));
	}
    
}
return wrap(R);
'

# here we compile the c function:
testf=cxxfunction(signature(a_="numeric",x_="numeric",beta_="numeric"),body=src,plugin="RcppGSL",includes=inc)
```

```
## ld: warning: directory not found for option '-L/usr/local/lib/gcc/i686-apple-darwin8/4.2.3/x86_64'
## ld: warning: directory not found for option '-L/usr/local/lib/gcc/i686-apple-darwin8/4.2.3'
```

```r

# here we call it
res1 <- testf(a_=a,x_=x,beta_=Beta)
res1[res1==1e9] = NA  # the value 1e9 stands for "not feasible"
persp(res1,theta=100,main="objective function",ticktype="detailed",x=a,y=x,zlab="objective function value")
```

![plot of chunk unnamed-chunk-3](figure/unnamed-chunk-3.png) 



### Setup the optimizer 

Now that we are more or less convinced that the objective returns sensible values, we can setup the optimizer. In terms of the above picture, for every value of `a` on the right axis, we want to pick different values of `x` and find the one that makes the objective function **smallest**. 


```r
src2 <- '
// map values from R
NumericVector a(a_);
NumericVector x(x_);
double beta = as<double>(beta_);
bool verbose = as<bool>(verbose_);
NumericVector ay = log(x);  // future values

int na = a.length();

NumericVector R(na);   // allocate a return vector

// optimizer setup
int status;
int iter=0,max_iter=100;
const gsl_min_fminimizer_type *T;
gsl_min_fminimizer *s;

int k = x.length();

gsl_interp_accel *accP = gsl_interp_accel_alloc() ;
gsl_interp *linP       = gsl_interp_alloc( gsl_interp_linear , k );
gsl_interp_init( linP, x.begin(), ay.begin(), k );

// parameter struct
struct my_f_params params = {a(0),beta,linP,accP,x,ay};

// gsl objective function
gsl_function F;
F.function = &my_obj;
F.params   = &params;

// bounds on choice variable
double low = x(0), up=x(k-1);
double m = 0.2;	//current minium

// create an fminimizer object of type "brent"
T = gsl_min_fminimizer_brent;
s = gsl_min_fminimizer_alloc(T);

printf("using %s method\\n", 
	   gsl_min_fminimizer_name(s));

for (int i=0;i < a.length(); i++){

	// load current grid values into struct
	// if interpolation schemes depend on current state
	// i, need to allocate linP(i) here and then load into
	// params

	struct my_f_params params = {a(i),beta,linP,accP,x,ay};
	F.params   = &params;

	// if optimization bounds change do that here

	low = x(0); 
	up=x(k-1);
	iter=0;	// reset iterator for each state
    m = 0.2; // reset minimum for each state

	// set minimzer on current obj function
	gsl_min_fminimizer_set( s, &F, m, low, up );

    // print results
	if (verbose){
	printf (" iter lower      upper      minimum      int.length\\n");
			printf ("%5d [%.7f, %.7f] %.7f %.7f\\n",
			iter, low, up,m, up - low);
	}
    
    // optimization loop
    do {
		iter++;
		status = gsl_min_fminimizer_iterate(s);
		m      = gsl_min_fminimizer_x_minimum(s);
		low    = gsl_min_fminimizer_x_lower(s);
		up     = gsl_min_fminimizer_x_upper(s);

		status = gsl_min_test_interval(low,up,0.001,0.0);

		if (status == GSL_SUCCESS && verbose){
		    Rprintf("Converged:\\n");
		}
		if (verbose){
		printf ("%5d [%.7f, %.7f] %.7f %.7f\\n",
				iter, low, up,m, up - low);
		}
	} 
	while (status == GSL_CONTINUE && iter < max_iter);
	R(i) = m;
}
gsl_min_fminimizer_free(s);
return wrap(R);
'
```


#### compiling the C function


```r
f2=cxxfunction(signature(a_="numeric",x_="numeric",beta_="numeric",verbose_="logical"),body=src2,plugin="RcppGSL",includes=inc)
```

```
## ld: warning: directory not found for option '-L/usr/local/lib/gcc/i686-apple-darwin8/4.2.3/x86_64'
## ld: warning: directory not found for option '-L/usr/local/lib/gcc/i686-apple-darwin8/4.2.3'
```

```r

# setup a wrapper to call. optional
myfun <- function(ass,sav,Beta,verbose){
  res <- f2(ass,sav,Beta,verbose)
	return(res)
}

myres <- myfun(a,x,Beta,verbose)
```


#### look at result

we can compare the analytical solution to the one we found by just plotting the optimal savings choices:


```r
plot(a, a * Beta/(1 + Beta), col = "red")
lines(a, myres)
legend("bottomright", legend = c("true", "approx"), lty = c(1, 1), col = c("red", 
    "black"))
```

![plot of chunk unnamed-chunk-6](figure/unnamed-chunk-6.png) 


##### How far is our solution from the closed form?


```r
print(max(a * Beta/(1 + Beta) - myres))
```

```
## [1] 0.05496
```

