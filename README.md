# Stripe-Subscriptions-api



## Introduction
This repository will help us to create subscription api's. PHP Language is use for this task. 
Laravel Cashier provides an expressive, fluent interface to [Stripe's](https://stripe.com) subscription billing services. It handles almost all of the boilerplate subscription billing code you are dreading writing. In addition to basic subscription management, Cashier can handle coupons, swapping subscription, subscription "quantities", cancellation grace periods, and even generate invoice PDFs.

## Official Documentation

Documentation for Cashier can be found on the [Laravel website](https://laravel.com/docs/billing).

## Features

1. Create New Subscription <br>
2. Cancel Subscription <br>
3. Update card info <br>
4. Swap subscription <br>
5. Get user subscription <br> 

# Steps which have to follow 

## 1.Install cachier package

```
composer require laravel/cashier
```
also this
```
php artisan vendor:publish --tag="cashier-migrations"
```
## 2. Run migration

```
php artisan migrate 
```
## 3. Modify the Model 


<h4>User.php</h4>

<b>Add the Billable trait to the User model.</b>

```
<?php
namespace App;
...
use Laravel\Cashier\Billable;
class User extends Authenticatable {
  use Billable;
  ...
}
```
## 4. make Stripe account

<p>Go to (https://dashboard.stripe.com/) and create an account.</p> <br>
<p> After creating an account you wiull get STRIPE_KEY and STRIPE_SECRET</p> <br>
<p> Copy them and past in .env file of project</p> <br>

```
STRIPE_KEY=pk_test_51KzzCSDa1SC*******************
STRIPE_SECRET=sk_test_51KzzC********************** 

```

## 5. Make a Trait with name of Subscription.php

```
<?php

namespace App\subscriptions;
use Illuminate\Http\Request;
use App\subscriptions\StripeSubscriptionInterface;
use App\libs\Response\GlobalApiResponse;
use Auth,stdClass,Carbon\Carbon;
use Stripe;
use Session;
use Exception;
use App\Helper\ErrorLog;
use App\Helper\LogActivity;
use App\Models\User;

trait Subscription {

    public function setSripeSecret(){
        return Stripe\Stripe::setApiKey(env('STRIPE_SECRET'));
    }


    public function setSripeKey(){

        return Stripe\Stripe::setApiKey(env('STRIPE_KEY'));
    }

    public function createSripeTokenAndPaymentMethod($data)
    {
      
        $stripe = new \Stripe\StripeClient(env('STRIPE_KEY'));

          $token=$stripe->tokens->create([
            'card' => [
              'number' => $data->card_no,
              'exp_month' => $data->exp_month,
              'exp_year' => $data->exp_year,
              'cvc' => $data->cvc,
            ],
          ]);
          
          $paymentMethod=$stripe->paymentMethods->create([
            'type' => 'card',
            'card' => [
                'number' => $data->card_no,
                'exp_month' => $data->exp_month,
                'exp_year' => $data->exp_year,
                'cvc' => $data->cvc,
              ],
          ]);

          return response()->json([
              'stripeToken'=>$token->id,
              'paymentMethod'=>$paymentMethod->id,
          ]);
          
    }

    public function getSubscribed($request)
    {
      
        Stripe\Stripe::setApiKey(env('STRIPE_SECRET'));

        $user=User::find($request->user_id);

        $stripe_plan=$request->stripe_plan;
        
        $data= new stdClass();
        $data->card_no=$request->card_no;
        $data->exp_month=$request->exp_month;
        $data->exp_year=$request->exp_year;
        $data->cvc=$request->cvc;

        if($user->subscribed($request->plan_name))
        {
            return GlobalApiResponse::getResponse(422, null, [
                'message'=>'User is already subscribed with this plan',
                'user' => $user,
                
            ]);
        }
        
        $response=$this->createSripeTokenAndPaymentMethod($data);
        
        try{

        
            if (is_null($user->stripe_id)) {
                $stripeCustomer = $user->createAsStripeCustomer();
            }

            \Stripe\Customer::createSource(
                $user->stripe_id,
                ['source' => $response->original['stripeToken']]
            );


            $subscription = $user->newSubscription($request->plan_name, $stripe_plan)
                ->create($response->original['paymentMethod'], 
                [
                    'email' => $user->email,  //user email
                ]
        );
            

            LogActivity::addToLog('New Subscription for user' . $user->name . ' Created Successfully!');
                return GlobalApiResponse::getResponse(200, null, [
                    'message'=>'Subscription has been created successfully',
                    'user' => $user,
                    'subscription' => $subscription,
                ]);

        } catch (Exception $ex) {
            ErrorLog::addToLog('Subscription', $ex);
            return GlobalApiResponse::getResponse(500, null, [
                'message' => $ex->getMessage()
            ]);
        }
    }


    public function cancelUserSubscription($request)
    {
        try
        {
            Stripe\Stripe::setApiKey(env('STRIPE_SECRET'));
            $user=User::find($request->user_id);

           
            /**
             * laravel cachier code to cancel the subscription
             */
            

            $cancelSubscription=$user->subscription($request->plan_name)->cancel();

            LogActivity::addToLog('Subscription for user' . $user->name.' has been cancelled Successfully!');
            return GlobalApiResponse::getResponse(200, null, [
                'message'=>'Subscription has been cancelled Successfully',
                'cancel_subscription'=>$cancelSubscription??'Already cancelled'
                
            ]);
        } catch (Exception $ex) {
            ErrorLog::addToLog('Subscription', $ex);
            return GlobalApiResponse::getResponse(500, null, [
                'message' => $ex->getMessage()
            ]);
        }
    }

    public function resumeSubscription($request)
    {
        
        try
        {
            $user=User::find($request->user_id);

            $subscription= $user->subscription($request->plan_name)->resume();

            LogActivity::addToLog('Subscription for user' . $user->name . 'has been resumed');
          
            return GlobalApiResponse::getResponse(200, null, [
              'message'=>'Subscriptions has been resumed',
              'resumed_subscription'=>$subscription
              
          ]);
        } catch (Exception $ex) {
            ErrorLog::addToLog('Subscription', $ex);
            return GlobalApiResponse::getResponse(500, null, [
                'message' => $ex->getMessage()
            ]);
        }
    }



    public function checkUserActiveSubscription($request)
    {

      try
      {
          $user=User::find($request->user_id);

          $subscriptions= $user->subscriptions()->active()->get();

          // LogActivity::addToLog('Subscription info get of user' . $user->name);
        
          return GlobalApiResponse::getResponse(200, null, [
            'message'=>'Active Subscriptions of user',
            'active_subscription'=>$subscriptions
            
        ]);
      } catch (Exception $ex) {
          ErrorLog::addToLog('Subscription', $ex);
          return GlobalApiResponse::getResponse(500, null, [
              'message' => $ex->getMessage()
          ]);
      }
      
    }

    public function updateCreditCardInfo($data)
    {
        try
        {
          $user=User::find($data->user_id);
          $response=$this->createSripeTokenAndPaymentMethod($data);
          

          $card=$user->updateDefaultPaymentMethod($response->original['paymentMethod']);

          LogActivity::addToLog('user' . $user->name . ' update card info Successfully!');

          return GlobalApiResponse::getResponse(200, null, [
                    'message'=>'credit card info has been updated successfully',
                    'user' => $user,
                    'card' => $card,
                ]);

        } catch (Exception $ex) {
          ErrorLog::addToLog('Subscription', $ex);
          return GlobalApiResponse::getResponse(500, null, [
              'message' => $ex->getMessage()
          ]);
      }
    }

    public function getRelatedPlanActiveSubscription($plan_name,$user_id)
    {
        $user=User::find($user_id);

        $subscriptions= $user->subscriptions($plan_name)->active()->get();
        if($subscriptions && count($subscriptions)>0)
        {
          return true;
        }
        return false;
    }
}

?>

```

## 6. Make a Controller

<b>I have made a controller with name of StripeSubscriptionController.php</b> <b> You can also make with custom name</b><br>

Note: If you make a controller with other name then don't forget to include trait in it.

<h6>My cotroller code is blow </h6>

```
<?php

namespace App\Http\Controllers\Subscription;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use  App\subscriptions\Subscription;
use stdClass;
use App\libs\Response\GlobalApiResponse;
use App\Models\User;
use Auth;
use Stripe;
use Session;
use Exception;
use Validator,Response;
use App\Helper\ErrorLog;
use App\Helper\LogActivity;
use Carbon\Carbon;
class StripeSubscriptionController extends Controller
{
    use Subscription;

    public function getNewSubscription(Request $request)
    {
        // dd($request->all());

        $validator = \Validator::make($request->all(), [
            'card_no'=>'required',
            'exp_month'=>'required',
            'exp_year'=>'required',
            'cvc'=>'required',
            'user_id'=>'required',
            'stripe_plan'=>'required',
            'plan_name'=>'required'
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors());
        }

        return $this->getSubscribed($request);
      
       

        
    }

    public function cancelSubscription(Request $request)
    {
        $validator = \Validator::make($request->all(), [
            'user_id'=>'required',
            'plan_name'=>'required'
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors());
        }

        return $this->cancelUserSubscription($request);
    }


    public function resumeUserSubscription(Request $request)
    {

        $validator = \Validator::make($request->all(), [
            'plan_name'=>'required',
            'user_id'=>'required',
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors());
        }

        $user=User::find($request->user_id);

        $subscription= $user->subscription($request->plan_name)->resume();

        return $this->resumeSubscription($request);
    }


    public function swapSubscription(Request $request)
    {
        
        $user=User::find($request->user_id);

        \Stripe\Stripe::setApiKey(env('STRIPE_SECRET'));

        $subscription = \Stripe\Subscription::retrieve($request->subscription_id);


        $new=\Stripe\Subscription::update($request->subscription_id, [
            'cancel_at_period_end' => false,
            'proration_behavior' => 'always_invoice',
            'items' => [
              [
                'id' => $subscription->items->data[0]->id,
                'price' => $request->plan_id,
              ],
            ],
          ]);
        $singleSubscription=$user->subscriptions($request->current_plan)->active()->get();

        $singleSubscription=$singleSubscription[0];
        $singleSubscription->name=$request->new_plan;
        $singleSubscription->stripe_price=$request->plan_id;

        $singleSubscription->save();
         

        return GlobalApiResponse::getResponse(200, null, [
            'message'=>'Subscription propogate',
            'subscription' => $subscription,
            'new'=>$new
            
        ]);

        // $timestamp = $user->subscription('Premium')->asStripeSubscription();

        // return Carbon::createFromTimeStamp($timestamp->current_period_end)->format('F jS, Y');;

       
    }


    public function updateCreditCard(Request $request)
    {
        $validator = \Validator::make($request->all(), [
            'card_no'=>'required',
            'exp_month'=>'required',
            'exp_year'=>'required',
            'cvc'=>'required',
            'user_id'=>'required',
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors());
        }

        $data= new stdClass();
        $data->card_no=$request->card_no;
        $data->exp_month=$request->exp_month;
        $data->exp_year=$request->exp_year;
        $data->cvc=$request->cvc;
        $data->user_id=$request->user_id;

        return $this->updateCreditCardInfo($data);
        
    }


    public function getUserActiveSubscription(Request $request)
    {
        $validator = \Validator::make($request->all(), [
            'user_id'=>'required',
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors());
        }

       return $this->checkUserActiveSubscription($request);
    }
}


```


## 7. Past this in api.php

```
Route::post('new-subscription', [StripeSubscriptionController::class, 'getNewSubscription']);
Route::post('cancel-subscription', [StripeSubscriptionController::class, 'cancelSubscription']);
Route::post('update-card', [StripeSubscriptionController::class, 'updateCreditCard']);

Route::get('user-active-subscription', [StripeSubscriptionController::class, 'getUserActiveSubscription']);

Route::post('resume-subscription',[StripeSubscriptionController::class, 'resumeUserSubscription']);
Route::post('swap-subscription',[StripeSubscriptionController::class, 'swapSubscription']);

```






