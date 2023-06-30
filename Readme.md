## Different types of relationship between data 
 - 1:1
 - 1:many
 - many : many
## Referencing V/S Embedding 
Refercing --> Normalised
Embedded ---> Denormalised

## when to referencing and  embedding
 - Relationship type
 - Data Access patterns
 - Data closeness

## types of referenccing 
- child refercing  (1: Few)
- parent referencing (1:many \ton)
- 2-way  referencing ( Many : Many )
<!--
<!-- 
Embedding because closeness
bookings     tours --Few : Few -----  locations
               P \
        parent refer\
            1: many
                      C \ 
        users  --1: many-   reviews
       P    parent refern  C 


tours --- few : few  --- users 
            Embedding 
                or 
        child referencing 

tours ---- bookings  -----user
    1:many              1:many
P           c c         P


-->

```

// GeoJSON
    startLocation : {
        type : {
            type : String , 
            default : 'Point, 
            enum : ['Point']
        }, 
        cordinates : [Number], 
        address : String, 
        description : String

    }, 
    locations :  [ 
        {
            type : {
            type : String , 
            default : 'Point, 
            enum : ['Point']
        }, 
        cordinates : [Number], 
        address : String, 
        description : String
        }
    ]

```
# Review Model 
## Referencing tour and user
```
reviewSchema = {
    review : {
        type : String, 
        required:[true, 'Review can not be empty!']
    }, 
    rating : {
        type : Number, 
        min : 1, 
        max : 5
    }, 
    createdAt: {
        type : Date, 
        default : Date.now
    }, 
    tour : {
        type : mongoose.Schema.ObjectId,
        ref : "tour",
        required: [true, 'Review must belong to a tour']
    }, 
    user : {
        type : mongoose.Schema.ObjectId,
        ref :"user",
        required: [true, 'Review must belong to a user']
    }


    {
        toJSON : {virtual : true}, 
        toObject : {virtual : true}
    }
}
```


## populate the tour and user
```
reviewSchema.pre(/^find/, (next)=> {
    this.populate({
        path : 'tour', 
        select : 'name', 
    }).populate({
        path: 'user', 
        select : 'name photo'
    })
})
```

## virtual populate 
``` 
// tour Model (child refeerencing)
    reviews : [
        {
            type : mongoose.Schema.ObjectId,
            ref :"reviews",
        }
    ]


// ---virtual populate 

tourSchema.virtual('reviews', {
    ref : 'Review' , 
    foreignField : 'true', 
    localField : '_id'
})

// getTour 
const tour = await Tour.findByid(id).populate('reviews');


//review Schema
reviewSchema.pre(/^find/, (next)=> {
    this.populate({
        path: 'user', 
        select : 'name photo'
    })
})

```


# Nested Routes
POST /tour/:tourid/reviews/:userid

```
//tour routes

router.route('/:tourid/reviews).
    .post(protect,restrictTo(user) , createReview)

createReview => {
    if(!req.body.tour) req.body.tour = req.params.tourId;
    if(!req.body.user) req.body.user= req.user.id;
}

```

## Nested routes in proper way
```
routes.use(/:tourid/reviews, reviewRouter)
```

### use this to merge routes(nested)
get access to tourid reviewController
```
const router = express.Router({ mergeParams: true });
```


getAllReviews 
```
let filter = {};
if(req.params.tourId) filter = {tour : req.params.tourId}

const reviews = await Review.find (filter);
```




# Factory code 
these are only allowed to do  for admin 
## Delete One
-  it returns controller function 

```
exports.deleteOne = Model =>
  catchAsync(async (req, res, next) => {
    const doc = await Model.findByIdAndDelete(req.params.id);

    if (!doc) {
      return next(new AppError('No document found with that ID', 404));
    }

    res.status(204).json({
      status: 'success',
      data: null
    });
});
```
- DeleteTour
```
exports.deleteTour = factory.deleteOne(Tour)
```


## Note 
Same for update and create 


# import data

```
// off the validators 
await User.create(users, {validateBeforeSave: false});
```

# improving Read performance 


handleFactory-> getALl
```
const doc  = await featurres.query.explain() 
```

## using indexes  

```
// 1 for Ascending Order 
// -1 for Descen.
tourSchema.index(price : 1 , ratingsAverage : -1)
tourSchema.index(slug : 1);
```


# improvements

## Rating Average 


### statics method 

```
reviewSchema.statics.calAverage = function(tourId) {
    const stats  = await this.aggregate([
        {
            $match : (tour: tourId)
        }, 
        {
            $group : {
                _id: '$tour', 
                nRating : {$sum : 1}, 
                avgRating : {$avg : $rating}
            }
        }
    ])

   if(stats.length > 0 ){
     Tour.findByIdAndUpdate (id, {
        ratingsquantityt: stats[0].nrating 
        avg : stats[0].avgRating
    });
   }else{
    Tour.findByIdAndUpdate (id, {
        ratingsquantityt: 0
        avg : 4.5
    });
   }

}

reviewSchema.post('save', function(){
    this.consructor.calaverage(this.tour);
})
```

### for update and Delete 
```
reviewSchema.pre(/^findOneAnd/, function(next) {
    this.r    = await this.findOne();
    next();
}) 

reviewSchema.post(/^findOneAnd/, asyncfunction(){
    await this.r.constructor.calAvgera(this.r.tour);
})


```

### preventing Duplicating review  
``` 
reviewSchmea.index({tour : 1 , user : 1}, {unique: true});
```

#### round off
```  
ratingsAverage : {
    set : val => Math.round(val*10) /10;
}
```


# Geospatial Queries 
``` 
router.route(/toors-within/:distance/center/:latlng/unit/:unit, toursController.getTourswithin)

getToursWithin = {
    const {distance, latlng,unit } = req.params;
    const [lat, lng]latlng.split(',');
    const radius = unit === 'mi' ? distance/3963.2 : distance/6378.1
    if(!lat || !lng) {
        next (new AppError('error'), 400);
    }

    const tours =  await Tour.find({
        startLocation : {
            $geoWithin :  { $centerSphere : [[lng, lat] , radius]}
        }
    });

    res.status(200).json({
        status : 'success', 
        data : {
            data : tours
        }
    })
}




tourSchema.index({startLocation : '2dsphere'})
```

## get distance 

``` 
router.route(/distances/:latlng/unit/:unit)
    .get(tourController.getDistances);


getDistances = {
    const { latlng, unit } = req.params;
  const [lat, lng] = latlng.split(',');

  const multiplier = unit === 'mi' ? 0.000621371 : 0.001;

  if (!lat || !lng) {
    next(
      new AppError(
        'Please provide latitutr and longitude in the format lat,lng.',
        400
      )
    );
  }

    const distances  = await Tour.aggreate([
        {
            $geoNear : {
                near: {
                    type : 'Point',
                    cordinates: [lng*1, lat * 1]
                }, 
                distanceField : 'distance', 
                distanceMultiplier: multiplier
            }, {
                $project : {
                    distance : 1, 
                    name : 1
                }
            }
        }
    ])
}
```"# Yatra" 
