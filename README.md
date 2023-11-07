<?php

namespace App\Http\Controllers;

use DB;
use PDF;
use Auth;
use Hash;
use Config;

use App\Cart;
use App\User;
use App\Products;
use App\AppImages;
use App\TimeSlots;
use Carbon\Carbon;
use App\OrderItems;
use App\CouponUsers;
use App\OrderDetails;
use App\productImages;
use App\UserAddresses;
use App\Helpers\Helper;
use App\productOptions;

use App\RegisteredUser;
use App\ProductCategories;
use Illuminate\Http\Request;
use Tzsk\Payu\Facade\Payment;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Session;

class UserController extends Controller
{
    //

    public function __construct()
    {
        $this->middleware('auth');
    }


    public function myaccount()
    {
        $regId = Auth::user()->id;
        // dd($regId);
        $registereduser = RegisteredUser::select(['fullName', 'mobileNumber', 'email'])->where('id', $regId)->first();
        return view('myaccount', compact('registereduser'));
    }

    public function smartbasket()
    {
        return view('smartbasket');
    }

    public function myorders()
    {
        return view('myorders');
    }

    public function addcheckoutaddress()
    {
        return view('addcheckoutaddress');
    }

    public function editprofile()
    {
        $regId = Auth::user()->id;
        $registereduser = RegisteredUser::select(['fullName', 'mobileNumber', 'email'])->where('id', $regId)->first();
        return view('editprofile', compact('registereduser'));
    }

    public function addresslist()
    {
        $regId = Auth::user()->id;
        // dd()
        $useraddresses = UserAddresses::where('userId', $regId)->where('status', '=', '1')->get();
        // dd($useraddresses);
        return view('addresslist', compact('useraddresses'));
    }

    public function changepassword()
    {
        return view('changepassword');
    }


    public function updateProfile(Request $request)
    {
        // dd($request);
        // return view('updateProfile');
        $this->validate($request, [

            'fullName' => 'required',
            'email' => 'required|string|email|max:255',


        ], [
            'fullName.required' => 'Please enter name',
            'email.required' => 'Please enter email',

        ]);

        $regId = Auth::user()->id;

        // $registereduser = RegisteredUser::where('email', )
        $registereduser = RegisteredUser::where('id', $regId)->first();

        $registereduserdata = RegisteredUser::where('email', $request->email)->where('id', '!=', $regId)->first();

        if (count($registereduserdata) == 0) {

            $registereduser->fullName = $request->fullName;
            $registereduser->email = $request->email;
            $saved = $registereduser->save();

            Session::flash('success', 'Changes has been updated successfully.');
            return redirect()->route('editprofile');
        } else {
            Session::flash('error', 'Email already in use!');
            return redirect()->route('editprofile');
        }
    }

    public function setAddressDefault(Request $request)
    {

        $regId = Auth::user()->id;

        $updateuseraddress = DB::table('user_addresses')->where('userId', $regId)->update(array('default_address' => '0'));

        $updatedefaultaddress = DB::table('user_addresses')->where('userId', $regId)->where('id', $request->addressId)->update(array('default_address' => '1'));

        if ($updatedefaultaddress) {
            echo 'success';
        } else {
            echo 'fail';
        }
    }

    public function deleteAddress(Request $request)
    {

        $useraddresses = UserAddresses::where('id', $request->addressId)->first();

        $useraddresses->status = '0';

        $deleteAddress = $useraddresses->save();

        if ($deleteAddress) {
            echo 'success';
        } else {
            echo 'fail';
        }
    }


    public function editaddress($addressId)
    {
        $addressIdData = base64_decode($addressId);
        $useraddresses = UserAddresses::where('id', $addressIdData)->first();
        return view('editaddress', compact('useraddresses'));
    }

    public function updateaddress(Request $request)
    {

        // $this->validate($request, [

        //     'fullName' => 'required',
        //     'address' => 'required',


        // ],[
        //    'fullName.required' => 'Please enter name',
        //    'address.required' => 'Please enter address',

        // ]);

        $addressId = base64_decode($request->addressId);



        $useraddresses = UserAddresses::where('id', $addressId)->first();

        if ($request->latitude != '' && $request->longitude != '') {
            $latitude = $request->latitude;
            $longitude = $request->longitude;
        } else {
            $latitude = $useraddresses->deliveryLatitude;
            $longitude = $useraddresses->deliveryLongitude;
        }

        $useraddresses->fullName = $request->fullName;
        $useraddresses->location = $request->location;
        $useraddresses->pincode = $request->pincode;
        $useraddresses->landmark = $request->landmark;
        $useraddresses->address = $request->address;
        $useraddresses->type = $request->type;
        $useraddresses->other_type = $request->other_type;
        $useraddresses->deliveryLatitude = $latitude;
        $useraddresses->deliveryLongitude = $longitude;
        $updated = $useraddresses->save();

        if ($updated) {
            Session::flash('success', 'Address changes has been updated successfully.');
            return redirect()->route('addresslist');
        } else {
            Session::flash('error', 'Error while updating address. Please try again!');
            return redirect()->route('editaddress', $request->addressId);
        }
    }

    public function addaddress()
    {
        return view('addaddress');
    }

    public function saveaddress(Request $request)
    {

        $this->validate($request, [

            'fullName' => 'required',
            'address' => 'required',


        ], [
            'fullName.required' => 'Please enter name',
            'address.required' => 'Please enter address',

        ]);

        $regId = Auth::user()->id;
        $mobileNumber = Auth::user()->mobileNumber;
        $email = Auth::user()->email;

        $useraddressesdata = UserAddresses::where('userId', $regId)->where('status', '1')->first();

        if (count($useraddressesdata) == 0) {

            $useraddresses = new UserAddresses();

            $useraddresses->userId = $regId;
            $useraddresses->fullName = $request->fullName;
            $useraddresses->mobileNumber = $mobileNumber;
            $useraddresses->email = $email;
            $useraddresses->location = $request->location;
            $useraddresses->pincode = $request->pincode;
            $useraddresses->landmark = $request->landmark;
            $useraddresses->address = $request->address;
            $useraddresses->type = $request->type;
            $useraddresses->other_type = $request->other_type;
            $useraddresses->default_address = '1';
            $useraddresses->deliveryLatitude = $request->latitude;
            $useraddresses->deliveryLongitude = $request->longitude;
        } else {

            $useraddresses = new UserAddresses();

            $useraddresses->userId = $regId;
            $useraddresses->fullName = $request->fullName;
            $useraddresses->mobileNumber = $mobileNumber;
            $useraddresses->email = $email;
            $useraddresses->location = $request->location;
            $useraddresses->pincode = $request->pincode;
            $useraddresses->landmark = $request->landmark;
            $useraddresses->address = $request->address;
            $useraddresses->type = $request->type;
            $useraddresses->other_type = $request->other_type;
            $useraddresses->deliveryLatitude = $request->latitude;
            $useraddresses->deliveryLongitude = $request->longitude;
        }


        $insert = $useraddresses->save();

        if ($insert) {
            Session::flash('success', 'New address has been added successfully.');
            return redirect()->route('addresslist');
        } else {
            Session::flash('error', 'Error while adding address. Please try again!');
            return redirect()->route('addaddress', $request->addressId);
        }
    }

    public function updatepassword(Request $request)
    {
        // $regId = Auth::user()->id;

        // if($request->password != $request->password_confirmation){
        //     Session::flash('error', 'New password and confirm password does not match.');
        //     return redirect()->route('changepassword');
        // }else{

        //     $rowExits = RegisteredUser::select(['id', 'userId'])->where('id', $regId)->where('password', $request->old_password)->first();

        //     if($rowExits){

        //         $updated = DB::table('registered_user')->where('id', $rowExits->id)->update(array('password'=> $request->password));

        //             if($updated ){

        //                 $updated = DB::table('users')->where('id', $rowExits->userId)->update(array('password'=> $request->password));
        //             }

        //         Session::flash('success', 'Password has been changed successfully.');
        //         return redirect()->route('changepassword');
        //     }else{
        //         Session::flash('error', 'Invalid old password.');
        //         return redirect()->route('changepassword');

        //     }
        // }

        $this->validate($request, [
            'old_password'     => 'required',
            'password' => 'required|string|min:8',
        ]);

        $data = $request->all();

        $user = RegisteredUser::find(auth()->user()->id);

        if ($request->password != $request->password_confirmation) {

            Session::flash('error', 'New password and confirm password does not match.');
            return redirect()->route('changepassword');
        } else {

            if (!Hash::check($data['old_password'], $user->password)) {

                return back()
                    ->with('error', 'Invalid old password');
            } else {
                // write code to update password
                $user->password = bcrypt($request->password);
                $user->save();


                Session::flash('success', 'Password updated successfully');
                return redirect()->route('changepassword');
            }
        }
    }

    public function getUserOrderListData(Request $request)
    {

        // dd($request->id);

        date_default_timezone_set('Asia/Kolkata');
        $regId = $request->regId;
        $page = $request->page;
        $cur_page = $page;
        $page -= 1;
        $per_page = 20;
        $last_id = $request->last_id;
        if (!isset($last_id) && $last_id == '') {
            $last_id = 0;
        }
        $start = $last_id * $per_page;
        $k = $start;

        $userDetails = RegisteredUser::select(['userId'])->where('id', $regId)->first();
        $userId = $userDetails->userId;

        $totalOrderCount = OrderDetails::select(['id'])->where('order_details.userId', $userId)->orderBy('id', 'DESC')->count();

        $results = DB::select("SELECT id,totalAmount,offerAmt,invoiceNo,invoiceNumber,orderStatus,dateTime,couponDiscount,expressDeliveryCost,deliverytimeSlot,deliveryDisplaySlot FROM order_details WHERE userId='" . $userId . "' OR regId='" . $regId . "' ORDER BY id DESC LIMIT $start, $per_page");

        if (count($results) > 0) {

            foreach ($results as $row) {
                $totalAmount = $row->totalAmount;
                $offerAmt = $row->offerAmt;
                $couponDiscount = $row->couponDiscount;
                if ($offerAmt > 0) {
                    $totalAmount = $totalAmount - $offerAmt;
                }
                if ($couponDiscount > 0) {
                    $totalAmount = $totalAmount - $couponDiscount;
                }
                if ($row->expressDeliveryCost != '' && $row->expressDeliveryCost > 0) {
                    $totalAmount = $totalAmount + $row->expressDeliveryCost;
                }
                if ($totalAmount < 0) {
                    $totalAmount = 0;
                }

                $k++;
                $processStatus = '';
                if ($row->orderStatus == 'Cancelled') {
                    $processStatus = 'Cancel';
                    $row->orderStatus = 'Cancel';
                }
                // if( ($row->orderStatus=='Cancel') && ($processStatus != 'Cancel')){
                //     $row->orderStatus = 'Processing';
                // }

                if ($row->orderStatus == 'Ready Pickup') {
                    $row->orderStatus = 'Processing';
                }
?>
<div class="col-md-6">
    <div class="orderListBox">
        <div class="orderBoxHeader">
            <h5><?php if ($row->invoiceNo != '') { ?> Order Tracking - <?php echo $row->invoiceNo ?> <?php } ?></h5>

            <div class="row titleBox no-gutters">
                <div class="col-4">
                    <span>Total</span>
                    <h4>&#8377; <?= number_format($totalAmount, 2) ?></h4>
                </div>
                <div class="col-4">
                    <span>Status</span>
                    <h4><?= $row->orderStatus ?></h4>
                </div>
                <div class="col-4">
                    <span>Ordered On</span>
                    <h4><?= \Carbon\Carbon::parse($row->dateTime)->format('M j, Y, g:i A') ?></h4>
                </div>
            </div>
        </div>
        <div class="processBox">

            <?php if ($processStatus == 'Cancel') { ?>
            <?php echo "<p class='text-center'>Your order has been canceled!</p>"; ?>
            <?php } else { ?>

            <div class="processinner d-flex p_relative justify-content-between">

                <div
                    class="progressBar <?php if ($row->orderStatus == 'Dispatched') { ?> shipped <?php } elseif ($row->orderStatus == 'Completed') { ?> delivered <?php } ?> ">
                </div>

                <div class="iconBox done d-inline-block p_relative" title-text="Processing">
                    <i class="fas fa-shopping-basket"></i>
                </div>

                <div class="iconBox <?php if ($row->orderStatus == 'Dispatched' || $row->orderStatus == 'Completed') { ?> done <?php } ?> d-inline-block p_relative"
                    title-text="Shipped">
                    <i class="fas fa-truck"></i>
                </div>

                <div class="iconBox <?php if ($row->orderStatus == 'Completed') { ?> done <?php } ?> d-inline-block p_relative"
                    title-text="Delivered">
                    <i class="fas fa-check"></i>
                </div>
            </div>
            <?php } ?>

        </div>
        <div class="col">
            <!-- <div class="row"> -->
            <p class="text-right"> <a class="viewOrderDetails"
                    href="<?= url('/') ?>/vieworderdetails/<?= base64_encode($row->id) ?>"
                    title="View Order Detail">View Details <i class="fa fa-caret-right" aria-hidden="true"></i></a></p>
            <!-- </div> -->
        </div>
    </div>
</div>
<?php if (($k % 2) == 0) { ?><div class="clearfix"></div> <?php } ?>
<script type="text/javascript">
var last_id = '<?= $last_id ?>';
last_id = parseInt(last_id) + parseInt(1);
</script>
<?php }
        } else { ?>
<script type="text/javascript">
var last_id = '-1';
</script>
<?php if ($totalOrderCount < 1) {
                echo "<p class='text-center'>New to Chitki.com?! Order now!</p>";
            }
        }
    }

    public function vieworderdetails($orderId)
    {

        $free_deliveryamount_limit = Config::get('constants.FREE_DELIVERYAMOUNT_LIMIT');

        $delivery_charge = Config::get('constants.DELIVERY_CHARGE');

        $orderIdData = base64_decode($orderId);

        $orderDetails = OrderDetails::select(['id', 'userId', 'invoiceNumber', 'invoiceNo', 'fullName', 'mobileNumber', 'email', 'address', 'location', 'pincode', 'landmark', 'note', 'subTotal', 'deliveryCost', 'totalAmount', 'dateTime', 'offerAmt', 'couponDiscount', 'giftOrder', 'expressDeliveryCost', 'deliverytimeSlot', 'deliveryDisplaySlot', 'merchantTxnId', 'paymentType', 'country', 'province', 'ship_fullName', 'ship_location', 'ship_country', 'ship_province', 'ship_address', 'companyName', 'ship_country_code', 'trn', 'country_code', 'ship_mobileNumber', 'tax', 'paymentType'])->where('id', $orderIdData)->first();

        $records = OrderItems::where('orderId', $orderDetails->id)->get();
        // dd($records[0]->order_variant_comb[0]->attr->attributeName);
        // dd($records[0]->variant_combination[0]->attr_value->attributeValue);

        return view('vieworderdetails', compact('orderDetails', 'records', 'free_deliveryamount_limit', 'delivery_charge'));
    }


    public function completeGiftOffersOrder(Request $request)
    {

        // date_default_timezone_set("Asia/Calcutta");
        date_default_timezone_set('Asia/Dubai');
        $dateTime  = date('Y-m-d H:i:s', time());
        $session_id = session()->get('session_id');
        $free_deliveryamount_limit = Config::get('constants.FREE_DELIVERYAMOUNT_LIMIT');
        $delivery_charge = Config::get('constants.DELIVERY_CHARGE');
        $business_offer_percent = Config::get('constants.BUSINESS_OFFER_PERCENT');
        $offer_percent = Config::get('constants.OFFER_PERCENT');

        // $query = "SELECT id, mobileNumber FROM ".USERS." WHERE mobileNumber='".$_POST['sendermobileNumber']."' ";

        $userData = User::select(['id', 'mobileNumber'])->where('mobileNumber', $request->sendermobileNumber)->first();

        if (count($userData) == 0) {

            $user = new User();


            $user->fullName = $request->fullName;
            $user->email = $request->email;
            $user->location = $request->location;
            $user->mobileNumber = $request->mobileNumber;
            $user->password = str_random(8);
            $user->address = $request->address;
            $user->dateTime = $dateTime;


            $userInsert = $user->save();

            $userId = $userInsert->id;
        } else {

            $userId = $userData->id;
        }

        if ($userId) {

            $orderdetails = new OrderDetails();

            // $data = array(
            $orderdetails->userId = $userId;
            $orderdetails->cartId = $session_id;
            $orderdetails->mobileNumber = $request->mobileNumber;
            $orderdetails->fullName = $request->fullName;
            $orderdetails->email = $request->email;
            $orderdetails->location = $request->location;
            $orderdetails->address = $request->address;
            $orderdetails->note = $request->note;
            $orderdetails->dateTime = $dateTime;
            $orderdetails->paymentType = 'Online';
            $orderdetails->onlinePaymentStatus = 'Pending';
            $orderdetails->giftOrder = 'Yes';
            $orderdetails->senderfullName = $request->senderfullName;
            $orderdetails->sendermobileNumber = $request->sendermobileNumber;
            $orderdetails->giftMessage = $request->giftMessage;

            $res = $orderdetails->save();

            $orderId = $orderdetails->id;

            if ($res) {

                $records = Cart::select(['id', 'productId', 'productOptionId', 'quantity', 'timeslotVal'])->where('cartId', $session_id)->get();

                $totalItems = 0;

                if (count($records) > 0) {

                    $subTotal = 0;
                    $grandtotal = 0;
                    $productTotal = 0;
                    $totalItems = 0;

                    foreach ($records as $record) {

                        $productData = Products::select(['id', 'productName', 'categoryId'])->where('id', $record['productId'])->first();

                        $productOption = productOptions::select(['id', 'productWeight', 'productUnit', 'productCost', 'productStock', 'orderCount', 'productOffer'])->where('id', $record['productOptionId'])->first();


                        if ((isset($productOption->productOffer)) && ($productOption->productOffer != '') && ($productOption->productOffer > 0) && (count($productOption->productOffer) > 0)) {
                            $offerPrice = round(($productOption->productOffer * $productOption->productCost) / 100);
                            if ($offerPrice < 1) {
                                $offerPrice = 1;
                            }
                            $productOption->productCost = ($productOption->productCost - $offerPrice);
                        }

                        $unitPrice = $productOption->productCost;
                        $productTotalPrice = $record['quantity'] * $productOption->productCost;

                        $orderedItems = new OrderItems();

                        $orderedItems->orderId = $orderId;
                        $orderedItems->productId = $record['productId'];
                        $orderedItems->productName = $productData->productName;
                        $orderedItems->productOptionId = $record['productOptionId'];
                        $orderedItems->productWeight = $productOption->productWeight;
                        $orderedItems->productUnit = $productOption->productUnit;
                        $orderedItems->unitPrice = $unitPrice;
                        $orderedItems->quantity = $record['quantity'];
                        $orderedItems->productTotalPrice = $productTotalPrice;
                        $orderedItems->timeslotVal = $record['timeslotVal'];

                        $orderItemInsert = $orderedItems->save();

                        if ($orderItemInsert) {

                            $orderCount = $productOption->orderCount + $record['quantity'];

                            // Stock Update ----

                            $productStock = 0;

                            $currentStock = $productOption->productStock;
                            $updatedStock = $currentStock - $record['quantity'];

                            if ($updatedStock >= 0) {
                                $productStock = $updatedStock;
                            } else {
                                // mail('sandesh@evol.co.in', 'CHITKI ALERT - '.$record['productId'], 'Stock goes minus : ProdId - '.$record['productId']);
                            }

                            //-----------------

                            // $orderCountDetail = array(
                            //     'orderCount' => $orderCount
                            // );

                            // $where_clause_ordercount = array(
                            //     'id' => $productOption->id
                            // );

                            // $db->update(PRODUCT_OPTIONS, $orderCountDetail, $where_clause_ordercount, 1);
                            $productOptionsUpdate = productOptions::where('id', $productOption->id)->first();

                            $productOptionsUpdate->orderCount = $orderCount;

                            $updatePO = $productOptionsUpdate->save();

                            $productTotal = $record['quantity'] * $productOption->productCost;
                            $totalItems += $record['quantity'];
                            $subTotal += $productTotal;
                        }
                    }



                    if ($orderItemInsert) {

                        $grandTotal = $subTotal;

                        $deliveryCost = 0;
                        $offerTotal = 0;

                        if ($subTotal < $free_deliveryamount_limit && $subTotal > 0) {
                            $deliveryCost = $delivery_charge;
                            $grandTotal += $delivery_charge;
                        }

                        $ORDERNUMBER = $orderId + 1000;

                        if (session()->get('businessUserOffer') == 'Yes') {
                            $offerTotal = round(($business_offer_percent / 100) * $subTotal);
                            if ($offerTotal < 1) {
                                $offerTotal = 1;
                            }
                            // unset($_SESSION['BUSINESS_OFFER_PERCENT']);
                            // unset($_SESSION['businessUserOfferAmount']);
                            session()->forget('BUSINESS_OFFER_PERCENT');
                            session()->forget('businessUserOfferAmount');
                        }

                        if (session()->get('offerAvailable') == 'Yes') {
                            $offerTotal = round(($offer_percent / 100) * $subTotal);
                            if ($offerTotal < 1) {
                                $offerTotal = 1;
                            }
                            // unset($_SESSION['offerAvailable']);
                            // unset($_SESSION['offerAmount']);
                            session()->forget('offerAvailable');
                            session()->forget('offerAmount');
                        }

                        if (session()->get('couponAvailable') != 'Yes') {
                            $couponDiscount = 0;
                        }

                        if (session()->get('couponAvailable') == 'Yes') {
                            $couponDiscount = session()->get('couponAmount');

                            $regId = Auth::user()->id;
                            // $regId = $oauth->authUser();
                            $couponusers = new CouponUsers();

                            $couponusers->regId = $regId;
                            $couponusers->couponId = session()->get('couponId');
                            $couponusers->couponCode = session()->get('couponCode');
                            $couponusers->dateTime = $dateTime;
                            $couponusers->userMobileNumber = $request->mobileNumber;

                            $couponUsersInsert = $couponusers->save();


                            // unset($_SESSION['couponAvailable']);
                            // unset($_SESSION['couponAmount']);
                            // unset($_SESSION['couponCode']);
                            // unset($_SESSION['couponId']);
                            session()->forget('couponAvailable');
                            session()->forget('couponAmount');
                            session()->forget('couponCode');
                            session()->forget('couponId');
                        }

                        // date_default_timezone_set("Asia/Calcutta");
                        date_default_timezone_set('Asia/Dubai');

                        // $updateOrderDetails = array(
                        //     'invoiceNumber' => $db->filter($ORDERNUMBER),
                        //     'subTotal' => $subTotal,
                        //     'deliveryCost' => $deliveryCost,
                        //     'totalAmount' => $grandTotal,
                        //     'offerAmt' => $offerTotal,
                        //     'couponDiscount' => $couponDiscount
                        // );

                        // $where_clause = array(
                        //     'id' => $orderId
                        // );

                        // $rs = $db->update(ORDER_DETAILS, $updateOrderDetails, $where_clause, 1);

                        $orderdetailsupdate = OrderDetails::where('id', $orderId)->first();

                        $orderdetailsupdate->invoiceNumber = $ORDERNUMBER;
                        $orderdetailsupdate->subTotal = $subTotal;
                        $orderdetailsupdate->deliveryCost = $deliveryCost;
                        $orderdetailsupdate->totalAmount = $grandTotal;
                        $orderdetailsupdate->offerAmt = $offerTotal;
                        $orderdetailsupdate->couponDiscount = $couponDiscount;

                        $rs = $orderdetailsupdate->save();
                    }
                }
            }
        }


        if ($rs) {
            // $_SESSION['orderinvoice'] = $ORDERNUMBER;
            // return $orderId;
            $useradddata = UserAddresses::where('userId', Auth::user()->id)->first();

            $merchantTxnId = date('ymdhms') . '_' . $ORDERNUMBER;

            $data = [
                'txnid' => $merchantTxnId, # Transaction ID.
                'amount' => $grandTotal, # Amount to be charged.
                'productinfo' => "CHITKI RETAILS PVT LTD",
                'firstname' => $useradddata->fullName, # Payee Name.
                'email' => $useradddata->email, # Payee Email Address.
                'phone' => $useradddata->mobileNumber, # Payee Phone Number.

            ];


            // $_SESSION['orderinvoice'] = $ORDERNUMBER;
            // return $orderId;

            Session::put('orderinvoice', $ORDERNUMBER);

            return Payment::with($orderdetailsupdate)->make($data, function ($then) {
                $then->redirectRoute('payumoneystatus');
            });
        } else {
            return false;
        }
    }


    public function addNewAddress(Request $request)
    {

        $regId = Auth::user()->id;
        $mobileNumber = Auth::user()->mobileNumber;
        $email = Auth::user()->email;

        $useraddressesdata = UserAddresses::where('userId', $regId)->where('status', '1')->first();

        // if(count($useraddressesdata)==1){
        //  $default_address = '1';
        // }
        if (count($useraddressesdata) == 0) {

            $useraddresses = new UserAddresses();

            $useraddresses->userId = $regId;
            $useraddresses->fullName = $request->fullName;
            $useraddresses->mobileNumber = $mobileNumber;
            $useraddresses->email = $email;
            $useraddresses->location = $request->location;
            $useraddresses->pincode = $request->pincode;
            $useraddresses->landmark = $request->landmark;
            $useraddresses->address = $request->address;
            $useraddresses->type = $request->type;
            $useraddresses->other_type = $request->other_type;
            $useraddresses->default_address = '1';
            $useraddresses->deliveryLatitude = $request->latitude;
            $useraddresses->deliveryLongitude = $request->longitude;
        } else {

            $useraddresses = new UserAddresses();

            $useraddresses->userId = $regId;
            $useraddresses->fullName = $request->fullName;
            $useraddresses->mobileNumber = $mobileNumber;
            $useraddresses->email = $email;
            $useraddresses->location = $request->location;
            $useraddresses->pincode = $request->pincode;
            $useraddresses->landmark = $request->landmark;
            $useraddresses->address = $request->address;
            $useraddresses->type = $request->type;
            $useraddresses->other_type = $request->other_type;
            $useraddresses->deliveryLatitude = $request->latitude;
            $useraddresses->deliveryLongitude = $request->longitude;
        }

        $saved = $useraddresses->save();

        return redirect()->route('checkout');
    }


    public function old_confirmPayment(Request $request)
    {

        // dd($request->location);
        // date_default_timezone_set("Asia/Calcutta");
        date_default_timezone_set('Asia/Dubai');
        $dateTime  = date('Y-m-d H:i:s', time());
        $session_id = session()->get('session_id');

        $free_deliveryamount_limit = Config::get('constants.FREE_DELIVERYAMOUNT_LIMIT');
        $delivery_charge = Config::get('constants.DELIVERY_CHARGE');
        $business_offer_percent = Config::get('constants.BUSINESS_OFFER_PERCENT');
        $offer_percent = Config::get('constants.OFFER_PERCENT');
        $checkStoreCatIds = Config::get('constants.STORE_CATEGORY_ID');

        if ($session_id == '') {
            return view('home');
        }

        $cartData = Cart::select(['id', 'categoryId'])->where('cartId', $session_id)->orderBy('id', 'DESC')->first();



        if ($request->paymenttype == 'COD') {

            $regId = Auth::user()->id;
            $mobileNumber = Auth::user()->mobileNumber;
            $email = Auth::user()->email;

            // $registereduser = RegisteredUser::select(['userId'])->where('id', $userId)->first();

            $userData = User::select(['id', 'mobileNumber'])->where('mobileNumber', $mobileNumber)->first();

            if (count($userData) == 0) {

                // $useraddresses = UserAddresses::where('id', $request->addressId)->first();

                $user = new User();

                $user->fullName = $request->fullName;
                $user->email = $email;
                $user->location = $request->location;
                $user->companyName = $request->companyName;
                // $user->pincode = $useraddresses->pincode;
                // $user->landmark = $useraddresses->landmark;
                $user->mobileNumber = $mobileNumber;
                $user->password = str_random(8);
                $user->address = $request->address;
                $user->dateTime = $dateTime;

                $userInsert = $user->save();

                $userId = $userInsert->id;
            } else {

                $userId = $userData->id;
            }

            if ($userId) {



                // $userupdateadress = UserAddresses::where('userId', $regId)->first();

                // $userupdateadress->default_address = '0';
                // $userupdateadress->save();
                // if($useraddresses->latitude=='' && $useraddresses->longitude=''){
                //     $useraddresses->latitude = $request->latitude;
                //     $useraddresses->longitude = $request->longitude;
                // }else{
                //    $useraddresses->latitude = $useraddresses->latitude;
                //    $useraddresses->longitude = $useraddresses->longitude;
                // }



                $updateuseraddress = DB::table('user_addresses')->where('userId', $regId)->update(array('default_address' => '0'));

                $updatedefaultaddress = DB::table('user_addresses')->where('userId', $regId)->where('id', $request->addressId)->update(array('default_address' => '1'));

                if ($request->latitude != '' && $request->longitude != '') {

                    $updateLatLong = DB::table('user_addresses')->where('userId', $regId)->where('id', $request->addressId)->update(array('deliveryLatitude' => $request->latitude, 'deliveryLongitude' => $request->longitude, 'location' => $request->location));
                }

                // $useraddresses = UserAddresses::where('id', $request->addressId)->first();

                $orderdetails = new OrderDetails();

                if ($request->expressDeliveryCost != '') {
                    $deliveryDateBy = date('Y-m-d H:i:s', strtotime('+1 hour +30 minutes'));
                } else {
                    $deliveryDateBy = $request->deliveryDateBy;
                }

                // if($cartData->categoryId=='73'){
                //     $fish = '1';
                // }else{
                //     $fish = '0';
                // }

                // if($cartData->categoryId=='66'){
                //     $mutton = '1';
                // }else{
                //     $mutton = '0';
                // }

                if ($request->giftaction == 'giftcompleteOrder') {
                    // $data = array(
                    $orderdetails->userId = $userId;
                    $orderdetails->regId = $regId;
                    $orderdetails->cartId = $session_id;
                    // $orderdetails->addressId = $request->addressId;
                    $orderdetails->mobileNumber = $mobileNumber;
                    $orderdetails->fullName = $request->fullName;
                    $orderdetails->email = $email;
                    $orderdetails->location = $request->location;
                    // $orderdetails->pincode = $useraddresses->pincode;
                    // $orderdetails->landmark = $useraddresses->landmark;
                    $orderdetails->address = $request->address;
                    // $orderdetails->note = $useraddresses->note;
                    // $orderdetails->deliveryLatitude = $useraddresses->deliveryLatitude;
                    // $orderdetails->deliveryLongitude = $useraddresses->deliveryLongitude;
                    $orderdetails->dateTime = $dateTime;
                    $orderdetails->giftOrder = 'Yes';
                    $orderdetails->senderfullName = $request->senderfullName;
                    $orderdetails->sendermobileNumber = $request->sendermobileNumber;
                    $orderdetails->giftMessage = $request->giftMessage;


                    $mobileNo = $request->sendermobileNumber;
                } else {

                    $orderdetails->userId = $userId;
                    $orderdetails->regId = $regId;
                    $orderdetails->cartId = $session_id;
                    // $orderdetails->addressId = $request->addressId;
                    $orderdetails->mobileNumber = $mobileNumber;
                    $orderdetails->fullName = $request->fullName;
                    $orderdetails->email = $email;
                    $orderdetails->location = $request->location;
                    $orderdetails->companyName = $request->companyName;
                    // $orderdetails->pincode = $useraddresses->pincode;
                    // $orderdetails->landmark = $useraddresses->landmark;
                    $orderdetails->address = $request->address;
                    // $orderdetails->note = $useraddresses->note;
                    // $orderdetails->deliveryLatitude = $useraddresses->deliveryLatitude;
                    // $orderdetails->deliveryLongitude = $useraddresses->deliveryLongitude;
                    $orderdetails->dateTime = $dateTime;
                    // $orderdetails->deliverytimeSlot = $request->deliverytimeSlot;
                    // $orderdetails->expressDeliveryCost = $request->expressDeliveryCost;
                    // $orderdetails->deliveryDisplaySlot = $request->deliveryDisplaySlot;
                    // $orderdetails->express = $request->express;
                    // $orderdetails->deliveryDateBy = $deliveryDateBy;
                    // $orderdetails->mutton = $mutton;
                    // $orderdetails->fish = $fish;
                    $mobileNo = $mobileNumber;
                }

                $res = $orderdetails->save();

                $orderId = $orderdetails->id;

                if ($res) {

                    $records = Cart::select(['id', 'productId', 'productOptionId', 'quantity', 'timeslotVal'])->where('cartId', $session_id)->where('quantity', '>', '0')->get();

                    $totalItems = 0;

                    if (count($records) > 0) {

                        $subTotal = 0;
                        $grandtotal = 0;
                        $productTotal = 0;
                        $totalItems = 0;

                        foreach ($records as $record) {

                            $productData = Products::select(['id', 'productName', 'categoryId'])->where('id', $record['productId'])->first();

                            $productOption = productOptions::select(['id', 'productWeight', 'productUnit', 'productCost', 'productStock', 'orderCount', 'productOffer', 'productDistributerPrice'])->where('id', $record['productOptionId'])->first();


                            if ((isset($productOption->productOffer)) && ($productOption->productOffer != '') && ($productOption->productOffer > 0) && (count($productOption->productOffer) > 0)) {
                                $offerPrice = round(($productOption->productOffer * $productOption->productCost) / 100);
                                if ($offerPrice < 1) {
                                    $offerPrice = 1;
                                }
                                $productOption->productCost = ($productOption->productCost - $offerPrice);
                            }

                            $unitPrice = $productOption->productCost;
                            $productTotalPrice = $record['quantity'] * $productOption->productCost;

                            $orderedItems = new OrderItems();

                            if ($request->giftaction == 'giftcompleteOrder') {

                                $orderedItems->orderId = $orderId;
                                $orderedItems->productId = $record['productId'];
                                $orderedItems->productName = $productData->productName;
                                $orderedItems->productOptionId = $record['productOptionId'];
                                $orderedItems->productWeight = $productOption->productWeight;
                                $orderedItems->productUnit = $productOption->productUnit;
                                $orderedItems->unitPrice = $unitPrice;
                                $orderedItems->quantity = $record['quantity'];
                                $orderedItems->productTotalPrice = $productTotalPrice;
                                $orderedItems->timeslotVal = $record['timeslotVal'];
                                $orderedItems->productCost = $productOption->productCost;
                                $orderedItems->productDistributerPrice = $productOption->productDistributerPrice;
                            } else {

                                $orderedItems->orderId = $orderId;
                                $orderedItems->productId = $record['productId'];
                                $orderedItems->productName = $productData->productName;
                                $orderedItems->productOptionId = $record['productOptionId'];
                                $orderedItems->productWeight = $productOption->productWeight;
                                $orderedItems->productUnit = $productOption->productUnit;
                                $orderedItems->unitPrice = $unitPrice;
                                $orderedItems->quantity = $record['quantity'];
                                $orderedItems->productTotalPrice = $productTotalPrice;
                                $orderedItems->productCost = $productOption->productCost;
                                $orderedItems->productDistributerPrice = $productOption->productDistributerPrice;
                            }

                            $orderItemInsert = $orderedItems->save();

                            if ($orderItemInsert) {

                                $orderCount = $productOption->orderCount + $record['quantity'];

                                // Stock Update ----
                                $productStock = 0;

                                $currentStock = $productOption->productStock;
                                $updatedStock = $currentStock - $record['quantity'];

                                if ($updatedStock >= 0) {
                                    $productStock = $updatedStock;
                                } else {
                                    // mail('santhosh@evol.co.in', 'CHITKI ALERT - '.$record['productId'], 'Stock goes minus : ProdId - '.$record['productId']);
                                }

                                //-----------------

                                $productOptionsUpdate = productOptions::where('id', $productOption->id)->first();

                                $productOptionsUpdate->orderCount = $orderCount;

                                $updatePO = $productOptionsUpdate->save();


                                $productTotal = $record['quantity'] * $productOption->productCost;
                                $totalItems += $record['quantity'];
                                $subTotal += $productTotal;
                            }
                        }

                        $expressDelivery = 'No';

                        if ($checkStoreCatIds != '') {

                            $storeCats = explode(',', $checkStoreCatIds);

                            if (in_array($cartData->categoryId, $storeCats)) {
                                $expressDelivery = 'Yes';
                            } else {
                                $expressDelivery = 'No';
                            }
                        }


                        if ($orderItemInsert) {

                            $grandTotal = $subTotal;

                            $deliveryCost = 0;
                            $offerTotal = 0;

                            if ($subTotal < $free_deliveryamount_limit && $subTotal > 0) {

                                if ($expressDelivery == 'Yes') {
                                    $delivery_charge = 0;
                                    $deliveryCost = 0;
                                } else {
                                    $deliveryCost = $delivery_charge;
                                }

                                // $deliveryCost = $delivery_charge;
                                $grandTotal += $delivery_charge;
                            }

                            $ORDERNUMBER = $orderId + 1000;

                            if (session()->get('businessUserOffer') == 'Yes') {
                                $offerTotal = round(($business_offer_percent / 100) * $subTotal);
                                if ($offerTotal < 1) {
                                    $offerTotal = 1;
                                }
                                // unset($_SESSION['BUSINESS_OFFER_PERCENT']);
                                // unset($_SESSION['businessUserOfferAmount']);
                                session()->forget('BUSINESS_OFFER_PERCENT');
                                session()->forget('businessUserOfferAmount');
                            }

                            if (session()->get('offerAvailable') == 'Yes') {
                                $offerTotal = round(($offer_percent / 100) * $subTotal);
                                if ($offerTotal < 1) {
                                    $offerTotal = 1;
                                }
                                // unset($_SESSION['offerAvailable']);
                                // unset($_SESSION['offerAmount']);
                                session()->forget('offerAvailable');
                                session()->forget('offerAmount');
                            }

                            if (session()->get('couponAvailable') != 'Yes') {
                                $couponDiscount = 0;
                            }

                            if (session()->get('couponAvailable') == 'Yes') {
                                $couponDiscount = session()->get('couponAmount');

                                $regId = Auth::user()->id;
                                // $regId = $oauth->authUser();
                                $couponusers = new CouponUsers();

                                $couponusers->regId = $regId;
                                $couponusers->couponId = session()->get('couponId');
                                $couponusers->couponCode = session()->get('couponCode');
                                $couponusers->dateTime = $dateTime;
                                $couponusers->userMobileNumber = $request->mobileNumber;

                                $couponUsersInsert = $couponusers->save();


                                // unset($_SESSION['couponAvailable']);
                                // unset($_SESSION['couponAmount']);
                                // unset($_SESSION['couponCode']);
                                // unset($_SESSION['couponId']);
                                session()->forget('couponAvailable');
                                session()->forget('couponAmount');
                                session()->forget('couponCode');
                                session()->forget('couponId');
                            }

                            // date_default_timezone_set("Asia/Calcutta");
                            date_default_timezone_set('Asia/Dubai');
                            // $invoiceNoOrderDetails = $db->get_row("SELECT invoiceNo FROM order_details WHERE invoiceNo IS NOT NULL AND TRIM(invoiceNo) <> '' ORDER BY id DESC LIMIT 1 ", true);
                            // $oldinvoice = substr($invoiceNoOrderDetails->invoiceNo,8);

                            /******* OLD CODE ********/

                            // if(date('d')=='01'||date('d')=='02'){
                            //   $oldinvoiceMonth = substr($invoiceNoOrderDetails->invoiceNo,6,2);
                            //   if(date('m')!=$oldinvoiceMonth){
                            //      $oldinvoice = 101000; //CH201703101001
                            //   }
                            // }
                            // $newinvoiceNo = $oldinvoice + 1;
                            // $newinvoiceNo = str_pad($newinvoiceNo, 3, '0', STR_PAD_LEFT);
                            // $yymm= date('Ym');
                            // $invoiceNo = "CH".$yymm.$newinvoiceNo;

                            /******* OLD CODE END ********/

                            //NEW INVOICE FORMAT 15052019

                            // $newinvoiceNo = $oldinvoice + 1;
                            // $newinvoiceNo = str_pad($newinvoiceNo, 6, '0', STR_PAD_LEFT);
                            // $yymm= '201920';
                            // $invoiceNo = "CH".$yymm.$newinvoiceNo;

                            $orderdetailsupdate = OrderDetails::where('id', $orderId)->first();

                            $orderdetailsupdate->invoiceNumber = $ORDERNUMBER;
                            $orderdetailsupdate->subTotal = $subTotal;
                            $orderdetailsupdate->deliveryCost = $deliveryCost;
                            $orderdetailsupdate->totalAmount = $grandTotal;
                            $orderdetailsupdate->offerAmt = $offerTotal;
                            $orderdetailsupdate->couponDiscount = $couponDiscount;

                            $rs = $orderdetailsupdate->save();
                        }
                    }
                }
            }


            if ($rs) {

                //$mobileNo = "7337844403";

                $orderdetailsupdatedata = OrderDetails::select(['mobileNumber', 'sendermobileNumber', 'deliverGuyId', 'orderStatus'])->where('invoiceNumber', $ORDERNUMBER)->where('orderStatus', '!=', 'Delivered')->first();

                if ($orderdetailsupdatedata->deliverGuyId == '') {

                    $message = Config::get('constants.SMS_MSG');

                    // $sentSms = $this->sendSMSViaMsgClub($mobileNo, $message);

                    if ($sentSms) {

                        $orderdetailsupdatedata->orderStatus = 'SMS sent';

                        $rs = $orderdetailsupdatedata->save();
                    }

                    // self::emailNotifications($orderId);
                }

                session()->forget('session_id');

                // session()->put('orderinvoice') = $ORDERNUMBER;
                Session::put('orderinvoice', $ORDERNUMBER);
                // return $ORDERNUMBER;
                return redirect()->route('thankyou');
            } else {
                // return false;
                return redirect()->route('checkout');
            }
        } elseif ($request->paymenttype == 'Online') {

            $regId = Auth::user()->id;
            $mobileNumber = Auth::user()->mobileNumber;
            $email = Auth::user()->email;
            // $registereduser = RegisteredUser::select(['userId'])->where('id', $userId)->first();
            $userData = User::select(['id', 'mobileNumber'])->where('mobileNumber', $mobileNumber)->first();

            if (count($userData) == 0) {

                // $useraddresses = UserAddresses::where('id', $request->addressId)->first();
                $user = new User();

                $user->fullName = $request->fullName;
                $user->email = $email;
                $user->location = $request->location;
                $user->companyName = $request->companyName;
                // $user->pincode = $useraddresses->pincode;
                // $user->landmark = $useraddresses->landmark;
                $user->mobileNumber = $request->mobileNumber;
                $user->password = str_random(8);
                $user->address = $request->address;
                $user->dateTime = $dateTime;

                $userInsert = $user->save();

                $userId = $userInsert->id;
            } else {

                $userId = $userData->id;
            }


            if ($userId) {

                // dd($request->addressId);

                // $useraddresses = UserAddresses::where('id', $request->addressId)->first();

                $updateuseraddress = DB::table('user_addresses')->where('userId', $regId)->update(array('default_address' => '0'));


                $updatedefaultaddress = DB::table('user_addresses')->where('userId', $regId)->where('id', $request->addressId)->update(array('default_address' => '1'));

                if ($request->latitude != '' && $request->longitude != '') {

                    $updateLatLong = DB::table('user_addresses')->where('userId', $regId)->where('id', $request->addressId)->update(array('deliveryLatitude' => $request->latitude, 'deliveryLongitude' => $request->longitude, 'location' => $request->location));
                }




                // $useraddresses = UserAddresses::where('id', $request->addressId)->first();

                $orderdetails = new OrderDetails();

                if ($request->expressDeliveryCost != '') {
                    $deliveryDateBy = date('Y-m-d H:i:s', strtotime('+1 hour +30 minutes'));
                } else {
                    $deliveryDateBy = $request->deliveryDateBy;
                }

                // if($cartData->categoryId=='73'){
                //     $fish = '1';
                // }else{
                //     $fish = '0';
                // }

                // if($cartData->categoryId=='66'){
                //     $mutton = '1';
                // }else{
                //     $mutton = '0';
                // }

                $orderdetails->userId = $userId;
                $orderdetails->regId = $regId;
                $orderdetails->cartId = $session_id;
                // $orderdetails->addressId = $request->addressId;
                $orderdetails->mobileNumber = $mobileNumber;
                $orderdetails->fullName = $request->fullName;
                $orderdetails->email = $email;
                $orderdetails->location = $request->location;
                // $orderdetails->pincode = $useraddresses->pincode;
                // $orderdetails->landmark = $useraddresses->landmark;
                $orderdetails->address = $request->address;
                // $orderdetails->note = $useraddresses->note;
                // $orderdetails->deliveryLatitude = $useraddresses->deliveryLatitude;
                // $orderdetails->deliveryLongitude = $useraddresses->deliveryLongitude;
                $orderdetails->dateTime = $dateTime;
                $orderdetails->paymentType = 'Online';
                $orderdetails->onlinePaymentStatus = 'Pending';
                // $orderdetails->deliverytimeSlot = $request->deliverytimeSlot;
                // $orderdetails->expressDeliveryCost = $request->expressDeliveryCost;
                // $orderdetails->deliveryDisplaySlot = $request->deliveryDisplaySlot;
                // $orderdetails->express = $request->express;
                // $orderdetails->deliveryDateBy = $deliveryDateBy;
                // $orderdetails->fish = $fish;
                // $orderdetails->mutton = $mutton;

                $res = $orderdetails->save();

                $orderId = $orderdetails->id;

                if ($res) {

                    $records = Cart::select(['id', 'productId', 'productOptionId', 'quantity', 'timeslotVal'])->where('cartId', $session_id)->get();


                    $totalItems = 0;

                    if (count($records) > 0) {

                        $subTotal = 0;
                        $grandtotal = 0;
                        $productTotal = 0;
                        $totalItems = 0;

                        foreach ($records as $record) {

                            $productData = Products::select(['id', 'productName', 'categoryId'])->where('id', $record['productId'])->first();

                            $productOption = productOptions::select(['id', 'productWeight', 'productUnit', 'productCost', 'productStock', 'orderCount', 'productOffer', 'productDistributerPrice'])->where('id', $record['productOptionId'])->first();

                            if ((isset($productOption->productOffer)) && ($productOption->productOffer != '') && ($productOption->productOffer > 0) && (count($productOption->productOffer) > 0)) {
                                $offerPrice = round(($productOption->productOffer * $productOption->productCost) / 100);
                                if ($offerPrice < 1) {
                                    $offerPrice = 1;
                                }
                                $productOption->productCost = ($productOption->productCost - $offerPrice);
                            }
                            $unitPrice = $productOption->productCost;
                            $productTotalPrice = $record['quantity'] * $productOption->productCost;

                            $orderedItems = new OrderItems();

                            $orderedItems->orderId = $orderId;
                            $orderedItems->productId = $record['productId'];
                            $orderedItems->productName = $productData->productName;
                            $orderedItems->productOptionId = $record['productOptionId'];
                            $orderedItems->productWeight = $productOption->productWeight;
                            $orderedItems->productUnit = $productOption->productUnit;
                            $orderedItems->unitPrice = $unitPrice;
                            $orderedItems->quantity = $record['quantity'];
                            $orderedItems->productTotalPrice = $productTotalPrice;
                            $orderedItems->productCost = $productOption->productCost;
                            $orderedItems->productDistributerPrice = $productOption->productDistributerPrice;

                            $orderItemInsert = $orderedItems->save();

                            if ($orderItemInsert) {

                                $orderCount = $productOption->orderCount + $record['quantity'];

                                // Stock Update ----

                                $productStock = 0;

                                $currentStock = $productOption->productStock;
                                $updatedStock = $currentStock - $record['quantity'];

                                if ($updatedStock >= 0) {
                                    $productStock = $updatedStock;
                                } else {
                                    // mail('sandesh@evol.co.in', 'CHITKI ALERT - '.$record['productId'], 'Stock goes minus : ProdId - '.$record['productId']);
                                }

                                //-----------------

                                $productOptionsUpdate = productOptions::where('id', $productOption->id)->first();

                                $productOptionsUpdate->orderCount = $orderCount;

                                $updatePO = $productOptionsUpdate->save();

                                $productTotal = $record['quantity'] * $productOption->productCost;
                                $totalItems += $record['quantity'];
                                $subTotal += $productTotal;
                            }
                        }

                        $expressDelivery = 'No';

                        if ($checkStoreCatIds != '') {

                            $storeCats = explode(',', $checkStoreCatIds);

                            if (in_array($cartData->categoryId, $storeCats)) {
                                $expressDelivery = 'Yes';
                            } else {
                                $expressDelivery = 'No';
                            }
                        }

                        if ($orderItemInsert) {

                            $grandTotal = $subTotal;

                            $deliveryCost = 0;
                            $offerTotal = 0;

                            if ($subTotal < $free_deliveryamount_limit && $subTotal > 0) {

                                if ($expressDelivery == 'Yes') {
                                    $delivery_charge = 0;
                                    $deliveryCost = 0;
                                } else {
                                    $deliveryCost = $delivery_charge;
                                }
                                // $deliveryCost = $delivery_charge;
                                $grandTotal += $delivery_charge;
                            }

                            $ORDERNUMBER = $orderId + 1000;
                            if (session()->get('businessUserOffer') == 'Yes') {
                                $offerTotal = round(($business_offer_percent / 100) * $subTotal);
                                if ($offerTotal < 1) {
                                    $offerTotal = 1;
                                }

                                session()->forget('BUSINESS_OFFER_PERCENT');
                                session()->forget('businessUserOfferAmount');
                            }
                            if (session()->get('offerAvailable') == 'Yes') {
                                $offerTotal = round(($offer_percent / 100) * $subTotal);
                                if ($offerTotal < 1) {
                                    $offerTotal = 1;
                                }
                                // unset($_SESSION['offerAvailable']);
                                // unset($_SESSION['offerAmount']);
                                session()->forget('offerAvailable');
                                session()->forget('offerAmount');
                            }

                            if (session()->get('couponAvailable') != 'Yes') {
                                $couponDiscount = 0;
                            }

                            if (session()->get('couponAvailable') == 'Yes') {
                                $couponDiscount = session()->get('couponAmount');
                                $regId = Auth::user()->id;

                                $couponusers = new CouponUsers();

                                $couponusers->regId = $regId;
                                $couponusers->couponId = session()->get('couponId');
                                $couponusers->couponCode = session()->get('couponCode');
                                $couponusers->dateTime = $dateTime;
                                $couponusers->userMobileNumber = $request->mobileNumber;

                                $couponUsersInsert = $couponusers->save();


                                // unset($_SESSION['couponAvailable']);
                                // unset($_SESSION['couponAmount']);
                                // unset($_SESSION['couponCode']);
                                // unset($_SESSION['couponId']);

                                session()->forget('couponAvailable');
                                session()->forget('couponAmount');
                                session()->forget('couponCode');
                                session()->forget('couponId');
                            }


                            $orderdetailsupdate = OrderDetails::where('id', $orderId)->first();

                            $orderdetailsupdate->invoiceNumber = $ORDERNUMBER;
                            $orderdetailsupdate->subTotal = $subTotal;
                            $orderdetailsupdate->deliveryCost = $deliveryCost;
                            $orderdetailsupdate->totalAmount = $grandTotal;
                            $orderdetailsupdate->offerAmt = $offerTotal;
                            $orderdetailsupdate->couponDiscount = $couponDiscount;

                            $rs = $orderdetailsupdate->save();
                        }
                    }
                }
            }


            if ($rs) {

                $useradddata = UserAddresses::where('id', $request->addressId)->first();

                $merchantTxnId = date('ymdhms') . '_' . $ORDERNUMBER;

                if ($offerTotal > 0) {
                    $grandTotal = $grandTotal - $offerTotal;
                }

                if ($couponDiscount > 0) {
                    $grandTotal = $grandTotal - $couponDiscount;
                }

                if ($request->expressDeliveryCost != '' && $request->expressDeliveryCost > 0) {
                    $grandTotal = $grandTotal + $request->expressDeliveryCost;
                }

                if ($grandTotal < 0) {
                    $grandTotal = 0;
                }

                $updated = OrderDetails::where('invoiceNumber', $ORDERNUMBER)->update(['merchantTxnId' => $merchantTxnId]);

                $data = [
                    'txnid' => $merchantTxnId, # Transaction ID.
                    'amount' => $grandTotal, # Amount to be charged.
                    'productinfo' => "CHITKI RETAILS PVT LTD",
                    'firstname' => $request->fullName, # Payee Name.
                    'email' => $email, # Payee Email Address.
                    'phone' => $mobileNumber, # Payee Phone Number.

                ];


                // $_SESSION['orderinvoice'] = $ORDERNUMBER;
                // return $orderId;

                Session::put('orderinvoice', $ORDERNUMBER);

                return Payment::with($orderdetailsupdate)->make($data, function ($then) {
                    $then->redirectRoute('payumoneystatus');
                });
            }
        } else {
            Session::flash('errors', 'Please select payment method!');
            return redirect()->route('checkout');
        }
    }

    public function emailNotifications($orderId)
    {

        $orderDetails = OrderDetails::where('id', $orderId)->first();

        $records = OrderItems::where('orderId', $orderDetails->id)->get();

        $invoiceNumber = $orderDetails->invoiceNo;

        $orderDate = \Carbon\Carbon::parse($orderDetails->dateTime)->format('F j, Y');

        $fromemail = 'orders@duboxx.com';
        $fromname = 'Duboxx - Online Packaging Store U.A.E';
        $toemail = $orderDetails->email;
        // $toemail = 'santhosh@evol.co.in';


        Mail::send('emails.thankyouMail', ['orderDetails' => $orderDetails, 'records' => $records], function ($message) use ($fromemail, $fromname, $invoiceNumber, $orderDate, $toemail) {

            $message->from($fromemail, $fromname);
            $message->to($toemail)->subject('Your Duboxx order receipt from ' . $orderDate . ' - #' . $invoiceNumber);
        });
    }


    public function emailNotificationsAdmin($orderId)
    {


        // $orderDetails = OrderDetails::select(['id', 'userId', 'invoiceNumber', 'fullName', 'mobileNumber', 'email', 'address', 'location', 'pincode', 'landmark', 'note', 'subTotal', 'deliveryCost', 'totalAmount', 'dateTime', 'offerAmt', 'couponDiscount', 'giftOrder', 'expressDeliveryCost', 'deliverytimeSlot', 'deliveryDisplaySlot', 'tax', 'paymentType'])->where('id', $orderId)->first();
        $orderDetails = OrderDetails::where('id', $orderId)->first();

        $records = OrderItems::where('orderId', $orderDetails->id)->get();

        $invoiceNo = $orderDetails->invoiceNo;

        $orderDate = \Carbon\Carbon::parse($orderDetails->dateTime)->format('F j, Y');

        $fromemail = 'orders@duboxx.com';
        $fromname = 'Duboxx - Online Packaging Store U.A.E';
        $toemail = 'orders@duboxx.com';
        // $toemail = 'samd123u@gmail.com';
        // $ccemail = 'santhosh@evol.co.in';
        // $bccemail = 'orders@duboxx.com';

        Mail::send('emails.orderReceiveMail', ['orderDetails' => $orderDetails, 'records' => $records], function ($message) use ($fromemail, $fromname, $invoiceNo, $orderDate, $toemail) {

            $message->from($fromemail, $fromname);
            $message->to($toemail)->subject('Duboxx: New customer order' . ' ' . ($invoiceNo) . ' - ' . $orderDate);
        });
    }


    public static function checkOutPage()
    {

        session()->forget('orderinvoice');

        $userId = Auth::user()->id;

        $express_time_limit = Config::get('constants.EXPRESS_TIME_LIMIT');

        $express_start_time = Config::get('constants.EXPRESS_START_TIME');

        $registereduser = RegisteredUser::where('id', $userId)->first();

        $userAddressesData = UserAddresses::where('userId', $userId)->where('status', '=', '1')->get();
        // dd($userAddressesData);

        $orderData = OrderDetails::select(['id', 'userId', 'addressId'])->where('userId', $registereduser->userId)->orderBy('id', 'DESC')->first();

        // dd($orderData);
        $orderIdData = '';
        if (count($orderData) > 0) {
            $orderIdData = $orderData->addressId;
        }

        $session_id = session()->get('session_id');

        if ($session_id == '') {
            return view('cart');
        }

        $cartData = Cart::select(['id', 'categoryId'])->where('cartId', $session_id)->orderBy('id', 'DESC')->first();

        $express_delivery_limit = Config::get('constants.EXPRESS_DELIVERY_LIMIT');
        $checkStoreCatIds = Config::get('constants.STORE_CATEGORY_ID');

        $expressDeliverySlot = TimeSlots::where('active', '1')->where('id', '6')->first();

        $records = Cart::select(['id', 'productId', 'productOptionId', 'quantity', 'timeslotVal'])->where('cartId', $session_id)->orderBy('id', 'DESC')->get();

        $expressDelivery = 'No';

        $expressAlert = AppImages::where('type', '=', 'expressalert')->where('active', '=', '1')->first();

        if (count($records) == 0) {
            return view('cart');
        }

        if (count($records) > 0) {

            $free_deliveryamount_limit = Config::get('constants.FREE_DELIVERYAMOUNT_LIMIT');

            $delivery_charge = Config::get('constants.DELIVERY_CHARGE');

            $subTotal = 0;
            $grandtotal = 0;
            $productTotal = 0;
            $totalItems = 0;
            $giftProduct = 'No';

            $giftCatIdsData = Config::get('constants.giftCatIds');

            $giftCatIds = explode(',', $giftCatIdsData);

            if ($checkStoreCatIds != '') {

                $storeCats = explode(',', $checkStoreCatIds);

                if (in_array($cartData->categoryId, $storeCats)) {
                    $expressDelivery = 'Yes';
                    $delivery_charge = 0;
                } else {
                    $expressDelivery = 'No';
                    $delivery_charge = $delivery_charge;
                }
            }


            foreach ($records as $record) {

                $productData = Products::select(['id', 'productName', 'categoryId'])->where('id', $record['productId'])->first();

                if (in_array($productData->categoryId, $giftCatIds)) {
                    $giftProduct = 'Yes';
                }

                $productImageData = productImages::select(['image'])->where('productId', $record['productId'])->first();

                $productOption = productOptions::select(['id', 'productWeight', 'productUnit', 'productCost', 'productStock', 'productOffer', 'active'])->where('id', $record['productOptionId'])->first();

                if ((isset($productOption->productOffer)) && ($productOption->productOffer != '') && ($productOption->productOffer > 0) && (count($productOption->productOffer) > 0)) {
                    $offerPrice = round(($productOption->productOffer * $productOption->productCost) / 100);
                    if ($offerPrice < 1) {
                        $offerPrice = 1;
                    }
                    $productOption->productCost = ($productOption->productCost - $offerPrice);
                    $productTotal = $record['quantity'] * $productOption->productCost;
                    $totalItems += $record['quantity'];
                    $subTotal += $productTotal;
                } else {

                    $productTotal = $record['quantity'] * $productOption->productCost;
                    $totalItems += $record['quantity'];
                    $subTotal += $productTotal;
                }
            } //end foreach

        } // endif

        $grandTotal = $subTotal;

        if ($subTotal < $free_deliveryamount_limit && $subTotal > 0) {
            $grandTotal += $delivery_charge;
        }



        ?>

<div class="col-md-8 col-sm-7">
    <div class="bg-white boxShadow px_15">


        <?php if (Session::has('errors')) { ?>
        <div class="alert alert-danger alert_warp alert-dismissible fade show" role="alert">
            <button type="button" class="close" data-dismiss="alert" aria-label="Close">
                <span aria-hidden="true">&times;</span>
            </button>
            <strong>Error:</strong> <?= Session::get('errors') ?>
        </div>
        <?php } ?>

        <div class="checkOutStepLinks">
            <ul class="nav nav-tabs" id="myTab" role="tablist">
                <li class="nav-item stepLinkItem current" id="addressTab_li">
                    <a class="nav-link active" id="address-tab" data-toggle="tab" href="#addressTab" role="tab"
                        aria-controls="addressTab" aria-selected="true">
                        <svg height="685pt" viewBox="-49 -21 685 685.3351" width="685pt"
                            xmlns="http://www.w3.org/2000/svg">
                            <path
                                d="m199.300781 173.8125c0-6.894531-5.589843-12.484375-12.488281-12.484375h-97.382812c-6.898438 0-12.484376 5.589844-12.484376 12.484375v72.414062c0 6.894532 5.585938 12.484376 12.484376 12.484376h97.382812c6.898438 0 12.488281-5.589844 12.488281-12.484376zm-97.386719 12.484375h72.414063v47.445313h-72.414063zm0 0" />
                            <path
                                d="m352.867188 173.8125c0-6.894531-5.589844-12.484375-12.484376-12.484375h-97.386718c-6.894532 0-12.484375 5.589844-12.484375 12.484375v72.414062c0 6.894532 5.589843 12.484376 12.484375 12.484376h97.386718c6.894532 0 12.484376-5.589844 12.484376-12.484376zm-97.382813 12.484375h72.414063v47.445313h-72.414063zm0 0" />
                            <path
                                d="m199.300781 308.652344c0-6.894532-5.589843-12.484375-12.488281-12.484375h-97.382812c-6.898438 0-12.484376 5.589843-12.484376 12.484375v72.414062c0 6.898438 5.585938 12.484375 12.484376 12.484375h97.382812c6.898438 0 12.488281-5.585937 12.488281-12.484375zm-97.386719 12.484375h72.414063v47.445312h-72.414063zm0 0" />
                            <path
                                d="m432.773438 287.226562v-132.757812c.121093-4.054688-1.785157-7.910156-5.082032-10.273438l-205.644531-141.988281c-4.25-2.949219-9.890625-2.945312-14.136719.019531l-204.398437 141.988282c-3.328125 2.347656-5.277344 6.1875-5.226563 10.253906v403.445312c0 6.890626 5.386719 12.929688 12.277344 12.929688h253.144531c33.648438 45.199219 86.726563 71.78125 143.078125 71.65625 98.429688 0 178.207032-80.195312 178.207032-178.628906 0-89.394532-66.070313-163.753906-152.21875-176.644532zm-409.515626-126.226562 191.667969-133.332031 192.875 133.355469v124.238281s-.777343-.019531-1.328125-.019531c-21.300781.035156-42.421875 3.917968-62.34375 11.457031-1.265625-.398438-2.585937-.582031-3.914062-.53125h-96.953125c-3.308594-.078125-6.507813 1.160156-8.902344 3.441406-2.390625 2.285156-3.773437 5.425781-3.847656 8.730469v72.558594c.0625 6.722656 5.390625 12.21875 12.109375 12.484374-4.582032 10.9375-8.125 22.277344-10.582032 33.878907h-58.871093c-20.683594-.007813-37.476563 16.71875-37.542969 37.402343v81.207032h-112.367188zm232.671876 207.582031h-.445313v-47.445312h44.460937c-17.625 12.792969-32.578124 28.914062-44.015624 47.445312zm-95.335938 177.289063v-81.207032c.054688-6.902343 5.679688-12.457031 12.574219-12.429687h55.484375c-.246094 3.746094-.382813 7.886719-.382813 11.777344.023438 28.464843 6.8125 56.523437 19.804688 81.859375zm246.191406 71.433594c-84.800781 0-153.546875-68.746094-153.546875-153.546876 0-84.796874 68.746094-153.542968 153.546875-153.542968 84.800782 0 153.546875 68.746094 153.546875 153.542968-.097656 84.761719-68.785156 153.449219-153.546875 153.546876zm0 0" />
                            <path
                                d="m408.042969 346.109375h-.019531c-26.65625.042969-52.128907 10.972656-70.527344 30.25-38.40625 39.28125-38.9375 103.035156-1.277344 142.285156l60.058594 71.589844c2.386718 2.792969 5.890625 4.378906 9.566406 4.332031h.007812c3.667969.058594 7.167969-1.511718 9.558594-4.296875l60.957032-72.539062c38.175781-39.417969 38.492187-103.167969.621093-142.273438-18.011719-18.785156-42.921875-29.386719-68.945312-29.347656zm50.132812 154.601563c-.222656.222656-.433593.457031-.636719.703124l-51.675781 61.5625-50.78125-60.648437c-.195312-.226563-.394531-.453125-.605469-.667969-28.667968-29.585937-28.277343-78.019531.90625-107.875 13.714844-14.429687 32.730469-22.632812 52.640626-22.707031h.011718c19.265625 0 37.695313 7.871094 51.015625 21.796875 28.664063 29.597656 28.273438 78.027344-.875 107.835938zm0 0" />
                            <path
                                d="m406.765625 386.648438c-30.425781 0-55.089844 24.660156-55.089844 55.085937s24.664063 55.089844 55.089844 55.089844c30.421875 0 55.085937-24.664063 55.085937-55.089844-.035156-30.410156-24.679687-55.054687-55.085937-55.085937zm0 85.203124c-16.636719 0-30.117187-13.484374-30.117187-30.117187s13.480468-30.117187 30.117187-30.117187c16.628906 0 30.113281 13.484374 30.113281 30.117187-.019531 16.625-13.488281 30.097656-30.113281 30.117187zm0 0" />
                        </svg>
                        Delivery Address</a>
                </li>

                <li class="nav-item stepLinkItem" id="paymentTab_li">
                    <a class="nav-link no-click" id="payment-tab" data-toggle="tab" href="#paymentTab" role="tab"
                        aria-controls="paymentTab" aria-selected="false">
                        <svg version="1.1" id="Layer_1" xmlns="http://www.w3.org/2000/svg"
                            xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" viewBox="0 0 512.16 512.16"
                            style="enable-background:new 0 0 512.16 512.16;" xml:space="preserve">
                            <g transform="translate(1 1)">
                                <g>
                                    <g>
                                        <path d="M272.067,336.147H408.6c5.12,0,8.533-3.413,8.533-8.533v-76.8c0-5.12-3.413-8.533-8.533-8.533H272.067
                                                        c-5.12,0-8.533,3.413-8.533,8.533v76.8C263.533,332.733,266.947,336.147,272.067,336.147z M280.6,259.347h119.467v59.733H280.6
                                                        V259.347z" />
                                        <path
                                            d="M41.667,225.213h68.267c5.12,0,8.533-3.413,8.533-8.533s-3.413-8.533-8.533-8.533H41.667
                                                        c-5.12,0-8.533,3.413-8.533,8.533S36.547,225.213,41.667,225.213z" />
                                        <path
                                            d="M144.067,225.213h68.267c5.12,0,8.533-3.413,8.533-8.533s-3.413-8.533-8.533-8.533h-68.267
                                                        c-5.12,0-8.533,3.413-8.533,8.533S138.947,225.213,144.067,225.213z" />
                                        <path d="M41.667,259.347H152.6c5.12,0,8.533-3.413,8.533-8.533s-3.413-8.533-8.533-8.533H41.667c-5.12,0-8.533,3.413-8.533,8.533
                                                        S36.547,259.347,41.667,259.347z" />
                                        <path d="M212.333,242.28h-25.6c-5.12,0-8.533,3.413-8.533,8.533s3.413,8.533,8.533,8.533h25.6c5.12,0,8.533-3.413,8.533-8.533
                                                        S217.453,242.28,212.333,242.28z" />
                                        <path
                                            d="M503.32,136.467c-5.973-7.68-13.653-11.947-23.04-12.8l-20.48-2.482V97.213V80.147c0-18.773-15.36-34.133-34.133-34.133
                                                        H33.133C14.36,46.013-1,61.373-1,80.147v17.067v68.267v187.733c0,15.413,10.357,28.518,24.453,32.718
                                                        c-0.43,17.262,12.631,32.248,30.161,33.842l394.24,44.373c0.853,0,2.56,0,3.413,0c17.067,0,32.427-12.8,34.133-29.013
                                                        l25.6-273.92C511.853,152.68,509.293,143.293,503.32,136.467z M16.067,105.747h426.667v22.187v29.013H16.067V105.747z
                                                         M33.133,63.08h392.533c9.387,0,17.067,7.68,17.067,17.067v8.533H16.067v-8.533C16.067,70.76,23.747,63.08,33.133,63.08z
                                                         M16.067,353.213v-179.2h426.667v179.2c0,9.387-7.68,17.067-17.067,17.067H33.987h-0.853
                                                        C23.747,370.28,16.067,362.6,16.067,353.213z M493.933,157.8l-25.6,273.92c-0.853,9.387-9.387,16.213-18.773,15.36
                                                        L56.173,402.707c-8.533-0.853-14.507-7.68-15.36-15.36h384.853c18.773,0,34.133-15.36,34.133-34.133V165.48v-28.16l19.627,1.707
                                                        c4.267,0,8.533,2.56,11.093,5.973C493.08,148.413,494.787,153.533,493.933,157.8z" />
                                    </g>
                                </g>
                            </g>
                        </svg>
                        Payment Option</a>
                </li>

            </ul>
        </div>
        <div class="checkoutContentBody p_relative">
            <form action="<?= route('confirmPayment') ?>" id="paymentFrom"
                onsubmit="checkPaymentSelected('paymentFrom'); return false;" method="POST">
                <?= csrf_field() ?>

                <input type="hidden" name="deliverytimeSlot" class="deliverytimeSlotField">
                <input type="hidden" name="deliveryDisplaySlot" class="deliverytimeDispField">
                <input type="hidden" name="expressDeliveryCost" class="deliveryCostField">
                <input type="hidden" name="deliveryDateBy" class="delvby">
                <input type="hidden" name="express" id="expressDeliveryStatus">

                <div class="tab-content" id="myTabContent">


                    <div class="tab-pane fade show active" id="addressTab" role="tabpanel"
                        aria-labelledby="address-tab">
                        <div class="deliveryAddress_list pt_30 pb-2">
                            <!-- <h3 id="address_error" class="py-0 mt-0 text-center" style="display: none;"></h3> -->
                            <ul class="list-unstyled row mb-0">

                                <!-- <div class="row"> -->
                                <div class="col-sm-6">
                                    <div class="form-group">
                                        <label>Full Name<span class="required">*</span></label>
                                        <input type="text" name="fullName" id="fullName" class="form-control" required
                                            onkeyup="hideWhenClicked('fullName_error');">
                                        <div id="fullName_error"></div>
                                    </div>

                                </div>

                                <div class="col-sm-6">
                                    <div class="form-group">
                                        <label>Company name (optional)</label>
                                        <input type="text" name="companyName" class="form-control" id="companyName">
                                    </div>
                                </div>
                                <div class="col-sm-6">
                                    <div class="form-group">
                                        <label>Town / City<span class="required">*</span></label>
                                        <input type="text" name="location" class="form-control" id="city" required=""
                                            onkeyup="hideWhenClicked('city_error');">
                                        <div id="city_error"></div>
                                    </div>
                                    <!--  <div class="form-group">
                                                            <label>Land Mark</label>
                                                            <input type="text" name="landmark" class="form-control">
                                                        </div> -->
                                </div>
                                <div class="col-sm-6">
                                    <div class="form-group">
                                        <label>Address<span class="required">*</span></label>
                                        <textarea name="address" class="form-control" id="address" required
                                            onkeyup="hideWhenClicked('address_error');"></textarea>
                                        <div id="address_error"></div>
                                    </div>
                                </div>
                                <!-- </div> -->

                            </ul>
                        </div>

                        <a class="btn tabSift_btn checkout_Btn" data-parent="#addressTab_li" data-point="#paymentTab_li"
                            href="#paymentTab">Proceed to Payment &nbsp;&nbsp;<i
                                class="fas fa-arrow-circle-right"></i></a>
                    </div>


                    <div class="tab-pane fade" id="paymentTab" role="tabpanel" aria-labelledby="payment-tab">

                        <div class="paymentOptionBox pt_30">
                            <h3 id="paymentTypeBoxMsg" class="text-center pt-0 mt-0" style="display: none;"></h3>
                            <h4>Select a payment method you prefer</h4>

                            <!-- <form> -->
                            <div class="row">
                                <div class="p_relative paymentRadioBox mb-1 col-sm-6">
                                    <input class="form-check-input" type="radio" name="paymenttype" id="cod" value="COD"
                                        onclick="selectPaymentOptions();">
                                    <label class="form-check-label" for="cod">
                                        <img src="<?= url('/') ?>/images/icons/cash-on-delivery.svg"
                                            class="paymentIcon">
                                        Cash on delivery
                                    </label>
                                </div>
                                <!-- <div class="p_relative paymentRadioBox mb-1 col-sm-6">
                                                <input class="form-check-input" type="radio" name="paymenttype" id="onliePayment" value="Online" onclick="selectPaymentOptions();">
                                                <label class="form-check-label" for="onliePayment">
                                                    <img src="<?= url('/') ?>/images/icons/online-payment.svg" class="paymentIcon">
                                                    Online Payment (Via Credit/Debit Card)
                                                </label>
                                            </div> -->
                            </div>
                            <!-- </form> -->
                        </div>
                        <button type="submit" class="btn placeOrderBtn checkout_Btn" id="btn-submit">Place order &amp;
                            Pay &nbsp;&nbsp;<i class="fas fa-arrow-circle-right"></i></button>
                    </div>
                </div>
            </form>

            <script type="text/javascript">
            $(document).ready(function() {

                $("#paymentFrom").submit(function(e) {

                    if (jQuery('input[name=paymenttype]:checked').length > 0) {

                        $("#btn-submit").attr("disabled", true);

                    } else {

                        $("#btn-submit").attr("disabled", false);

                    }

                });
            });
            </script>
            <!-- address add modal form -->
            <div class="modal fade" id="addAddressModal" tabindex="-1" role="dialog"
                aria-labelledby="addAddressModalLabel" aria-hidden="true">
                <div class="modal-dialog modal-dialog-centered modal-lg" role="document">
                    <div class="modal-content productPopup_box">
                        <div class="modal-header">
                            <h5 class="modal-title" id="addAddressModalLabel">Add Address</h5>
                            <button type="button" class="close" data-dismiss="modal" aria-label="Close">
                                <span aria-hidden="true">&times;</span>
                            </button>
                        </div>
                        <div class="modal-body">
                            <div class="addAdressModal_body">
                                <form action="<?= route('addNewAddress') ?>" method="POST">
                                    <?= csrf_field() ?>
                                    <div class="row">
                                        <div class="col-sm-6">
                                            <div class="form-group">
                                                <label>Full Name<span class="required">*</span></label>
                                                <input type="text" name="fullName" class="form-control" required>
                                            </div>
                                        </div>

                                        <div class="col-sm-6">
                                            <div class="form-group">
                                                <label>Location</label>
                                                <input type="text" name="location" class="form-control" id="location">
                                            </div>
                                        </div>
                                        <div class="col-sm-6">
                                            <div class="form-group">
                                                <label>Pincode</label>
                                                <input type="text" name="pincode" class="form-control"
                                                    pattern="[1-9][0-9]{5}" title="Enter valid 6 digit pincode">
                                            </div>
                                            <div class="form-group">
                                                <label>Land Mark</label>
                                                <input type="text" name="landmark" class="form-control">
                                            </div>
                                        </div>
                                        <div class="col-sm-6">
                                            <div class="form-group">
                                                <label>Address<span class="required">*</span></label>
                                                <textarea name="address" class="form-control" required></textarea>
                                            </div>
                                        </div>
                                    </div>
                                    <div class="addAdresModalBtn_box text-right">
                                        <button type="submit" class="btn submit_btn">Add</button>
                                    </div>
                                </form>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div class="fllterButton_wrapper pb-4">
            <a href="<?= url('/') ?>/cart" class="btn viewCart_btn">View cart</a>
        </div>
    </div>
</div>



<div class="col-md-4 col-sm-5">
    <div class="checkout_billBox">
        <h3 class="bill_title">Order Summary</h3>
        <div class="bill_summaryBox">
            <div class="row_line d-flex justify-content-between">
                <h4>Cart Value</h4>
                <h5>AED <?= number_format($subTotal, 2) ?></h5>
            </div>
            <?php
                    // $grandTotal = $subTotal;
                    if ($subTotal < $free_deliveryamount_limit && $subTotal > 0) { ?>
            <?php if ($expressDelivery == 'No') { ?>
            <div class="row_line d-flex justify-content-between">
                <h4>Delivery Charge</h4>
                <h5>AED <?= number_format($delivery_charge, 2) ?></h5>
            </div>
            <?php } ?>

            <?php
                        // $grandTotal+=$delivery_charge;
                    } ?>
            <span class="expressDelSlot" style="display: none;">
                <div class="row_line d-flex justify-content-between expressDeliveryChargeMsg">

                </div>
            </span>

            <hr class="mt_10 mb-0">

            <div class="row_line d-flex justify-content-between">
                <h4>Total Payable</h4>
                <h5 id="totalAmountToPay">AED <?= number_format($grandTotal, 2) ?></h5>
                <input type="hidden" id="totalPayableAmt" value="<?= $grandTotal ?>">
            </div>
            <!-- <h4 class="selectedSlotTimeOption deliverySlotMsg" style="display: none;"></h4> -->
        </div>
        <?php
                if ($subTotal < $free_deliveryamount_limit && $subTotal > 0) { ?>
        <?php if ($expressDelivery == 'No') { ?>
        <div class="infoBox">

            Shop for AED. <?= $free_deliveryamount_limit ?> or more for FREE delivery
        </div>
        <?php } ?>
        <?php } ?>
    </div>
</div>


<?php
    }


    public function editcheckoutaddress($addressId)
    {
        // dd($addressId);
        $addressIdData = base64_decode($addressId);
        $useraddresses = UserAddresses::where('id', $addressIdData)->first();
        return view('editcheckoutaddress', compact('useraddresses'));
    }

    public function updatecheckoutaddress(Request $request)
    {

        $addressId = base64_decode($request->addressId);

        $useraddresses = UserAddresses::where('id', $addressId)->first();

        if ($request->latitude != '' && $request->longitude != '') {
            $latitude = $request->latitude;
            $longitude = $request->longitude;
        } else {
            $latitude = $useraddresses->deliveryLatitude;
            $longitude = $useraddresses->deliveryLongitude;
        }

        $useraddresses->fullName = $request->fullName;
        $useraddresses->location = $request->location;
        $useraddresses->pincode = $request->pincode;
        $useraddresses->landmark = $request->landmark;
        $useraddresses->address = $request->address;
        $useraddresses->type = $request->type;
        $useraddresses->other_type = $request->other_type;
        $useraddresses->deliveryLatitude = $latitude;
        $useraddresses->deliveryLongitude = $longitude;
        $updated = $useraddresses->save();

        if ($updated) {
            Session::flash('success', 'Address changes has been updated successfully.');
            return redirect()->route('checkout');
        } else {
            Session::flash('error', 'Error while updating address. Please try again!');
            return redirect()->route('editcheckoutaddress', $request->addressId);
        }
    }


    public function confirmAddress(Request $request)
    {

        $regId = Auth::user()->id;

        // dd($request->shipaddresscheck);

        $userAddressesData = UserAddresses::where('userId', $regId)->first();

        if (count($userAddressesData) > 0) {

            if ($request->shipaddresscheck == 'on') {
                $userAddressesData->userId = $regId;
                $userAddressesData->fullName = $request->fullName;
                $userAddressesData->country_code = $request->country_code;
                $userAddressesData->mobileNumber = $request->mobileNumber;
                $userAddressesData->email = $request->email;
                $userAddressesData->trn = $request->trn;
                $userAddressesData->company_name = $request->company_name;
                $userAddressesData->location = $request->location;
                $userAddressesData->address = $request->address;
                $userAddressesData->country = $request->country;
                $userAddressesData->province = $request->province;
                $userAddressesData->ship_fullName = $request->ship_fullName;
                $userAddressesData->ship_country_code = $request->ship_country_code;
                $userAddressesData->ship_mobileNumber = $request->ship_mobileNumber;
                $userAddressesData->ship_location = $request->ship_location;
                $userAddressesData->ship_address = $request->ship_address;
                $userAddressesData->ship_country = $request->ship_country;
                $userAddressesData->ship_province = $request->ship_province;
                $userAddressesData->customer_note = $request->customer_note;
                $saved = $userAddressesData->save();
            } else {

                $userAddressesData->userId = $regId;
                $userAddressesData->fullName = $request->fullName;
                $userAddressesData->country_code = $request->country_code;
                $userAddressesData->mobileNumber = $request->mobileNumber;
                $userAddressesData->email = $request->email;
                $userAddressesData->trn = $request->trn;
                $userAddressesData->company_name = $request->company_name;
                $userAddressesData->location = $request->location;
                $userAddressesData->address = $request->address;
                $userAddressesData->country = $request->country;
                $userAddressesData->province = $request->province;
                $userAddressesData->ship_fullName = $request->fullName;
                $userAddressesData->ship_country_code = $request->country_code;
                $userAddressesData->ship_mobileNumber = $request->mobileNumber;
                $userAddressesData->ship_location = $request->location;
                $userAddressesData->ship_address = $request->address;
                $userAddressesData->ship_country = $request->country;
                $userAddressesData->ship_province = $request->province;
                $userAddressesData->customer_note = $request->customer_note;
                $saved = $userAddressesData->save();
            }
        } else {

            $useraddresses = new UserAddresses();

            if ($request->shipaddresscheck == 'on') {

                $useraddresses->userId = $regId;
                $useraddresses->fullName = $request->fullName;
                $useraddresses->country_code = $request->country_code;
                $useraddresses->mobileNumber = $request->mobileNumber;
                $useraddresses->email = $request->email;
                $useraddresses->trn = $request->trn;
                $useraddresses->company_name = $request->company_name;
                $useraddresses->location = $request->location;
                $useraddresses->address = $request->address;
                $useraddresses->country = $request->country;
                $useraddresses->province = $request->province;
                $useraddresses->ship_fullName = $request->ship_fullName;
                $useraddresses->ship_country_code = $request->ship_country_code;
                $useraddresses->ship_mobileNumber = $request->ship_mobileNumber;
                $useraddresses->ship_location = $request->ship_location;
                $useraddresses->ship_address = $request->ship_address;
                $useraddresses->ship_country = $request->ship_country;
                $useraddresses->ship_province = $request->ship_province;
                $useraddresses->customer_note = $request->customer_note;
                $saved = $useraddresses->save();
            } else {

                $useraddresses->userId = $regId;
                $useraddresses->fullName = $request->fullName;
                $useraddresses->country_code = $request->country_code;
                $useraddresses->mobileNumber = $request->mobileNumber;
                $useraddresses->email = $request->email;
                $useraddresses->trn = $request->trn;
                $useraddresses->company_name = $request->company_name;
                $useraddresses->location = $request->location;
                $useraddresses->address = $request->address;
                $useraddresses->country = $request->country;
                $useraddresses->province = $request->province;
                $useraddresses->ship_fullName = $request->fullName;
                $useraddresses->ship_country_code = $request->country_code;
                $useraddresses->ship_mobileNumber = $request->mobileNumber;
                $useraddresses->ship_location = $request->location;
                $useraddresses->ship_address = $request->address;
                $useraddresses->ship_country = $request->country;
                $useraddresses->ship_province = $request->province;
                $useraddresses->customer_note = $request->customer_note;
                $saved = $useraddresses->save();
            }
        }


        if ($saved) {
            // Session::flash('success', 'Brand has been deleted successfully!');
            return redirect()->route('proceedToPayment');
        } else {
            return redirect()->route('checkout');
        }
    }

    public function proceedToPayment()
    {
        $dubai_delivery_charge = Config::get('constants.DUBAI_DELIVERY_CHARGE');
        $sharjah_delivery_charge = Config::get('constants.SHARJAH_DELIVERY_CHARGE');
        $other_country_delivery_charge = Config::get('constants.OTHER_COUNTRY_DELIVERY_CHARGE');
        return view('proceedToPayment', compact('dubai_delivery_charge', 'sharjah_delivery_charge', 'other_country_delivery_charge'));
    }

    public static function paymentProceedPage()
    {

        $session_id = session()->get('session_id');

        if ($session_id == '') {
            return view('home');
        }

        $regId = Auth::user()->id;

        $cartData = Cart::select(['id', 'categoryId'])->where('cartId', $session_id)->orderBy('id', 'DESC')->first();

        $checkStoreCatIds = Config::get('constants.STORE_CATEGORY_ID');

        $records = Cart::select(['id', 'productId', 'productOptionId', 'quantity', 'timeslotVal'])->where('cartId', $session_id)->orderBy('id', 'DESC')->get();

        $userAddress = UserAddresses::where('userId', $regId)->first();

        $dubai_deliveryamount_limit = Config::get('constants.DUBAI_DELIVERY_LIMIT');
        $dubai_delivery_charge = Config::get('constants.DUBAI_DELIVERY_CHARGE');

        $sharjah_deliveryamount_limit = Config::get('constants.SHARJAH_DELIVERY_LIMIT');
        $sharjah_delivery_charge = Config::get('constants.SHARJAH_DELIVERY_CHARGE');

        $other_country_deliveryamount_limit = Config::get('constants.OTHER_COUNTRY_DELIVERY_LIMIT');
        $other_country_delivery_charge = Config::get('constants.OTHER_COUNTRY_DELIVERY_CHARGE');

        $western_country_delivery_charge = Config::get('constants.WESTERN_COUNTRY_DELIVERY_CHARGE');

        $dubai_express_delivery_cost = Config::get('constants.DUBAI_EXPRESS_DELIVERY_COST');
        $other_express_delivery_cost = Config::get('constants.OTHER_EXPRESS_DELIVERY_COST');

    ?>

<div class="cartPage_wrapper">
    <!-- <h3 class="pageTitle_text">Proceed to Payment</h3> -->
    <h3 class="pageTitle_text">Confirm Order</h3>

    <?php
            if (count($records) > 0) {
                $subTotal = 0;
                $grandtotal = 0;
                $productTotal = 0;
                $totalItems = 0;
                $emptyStockExits = 'No';
                $emptyStockCartIds = '';
                $expressDelivery = 'No';
            ?>

    <div class="cartItems_tableBox">
        <div class="cart_table">
            <div class="cart_table_row cart_table_head_grup">
                <div class="cart_table_head">Item</div>
                <div class="w_340 cart_table_head">Item Description</div>
                <div class="cart_table_head">Price</div>
                <div class="cart_table_head">Qty</div>
                <div class="cart_table_head">Total</div>

            </div>
            <?php
                        foreach ($records as $record) {
                            $stockEmptyMsg = '';

                            $productData = Products::select(['id', 'productName', 'disableQty', 'active'])->where('id', $record['productId'])->first();

                            $productseoUrl = Helper::seoUrl(trim($productData->productName));

                            $productImageData = productImages::select(['image'])->where('productId', $record['productId'])->first();

                            /* calculate total */

                            $productOption = productOptions::select(['id', 'productWeight', 'productUnit', 'productCost', 'productStock', 'productOffer', 'active'])->where('id', $record['productOptionId'])->first();


                            $productTotal = $record['quantity'] * $productOption->productCost;
                            $totalItems += $record['quantity'];
                            $subTotal += $productTotal;

                            /* check stock */

                            $productOptionStockCheck = productOptions::select(['productStock'])->where('active', '1')->where('productId', $record['productId'])->where('productWeight', $productOption->productWeight)->where('productUnit', $productOption->productUnit)->first();

                            if ($productOptionStockCheck->productStock < 1 || $productOption->productStock < 1 || $productData->active == 0) {
                                $stockEmptyMsg = '<br/><span class="cartItemStockmsg">Out of stock</span>';

                                $emptyStockCartIds .= $record['id'] . ',';
                            }


                        ?>
            <div class="cart_table_row">
                <div class="cart_table_data cartProduct_img">
                    <!-- <img src="images/products/vegitables.png" class="img-fluid cartProduct_img"> -->
                    <a href="<?= url('product') ?>/<?= $productseoUrl ?>-<?= base64_encode($productData->id) ?>">
                        <?php if (isset($productImageData->image) && $productImageData->image != '') { ?>
                        <img src="<?= url('/') ?>/productFiles/images/thumb/<?= $productImageData->image ?>"
                            title="Duboxx - <?= $productData->productName ?>"
                            alt="Duboxx - Buy Boxes Online Dubai - Duboxx Largest Online Packaging Store UAE"
                            class="w-100">
                        <?php } else { ?>
                        <img src="<?= url('/') ?>/productFiles/images/thumb/default.png"
                            title="Duboxx - Buy Boxes Online Dubai - Duboxx Largest Online Packaging Store UAE"
                            alt="Duboxx - Buy Boxes Online Dubai - Duboxx Largest Online Packaging Store UAE"
                            class="w-100">
                        <?php } ?>
                    </a>
                </div>
                <div class="cart_table_data card_prodDiscripsBox">
                    <div>
                        <a href="<?= url('product') ?>/<?= $productseoUrl ?>-<?= base64_encode($productData->id) ?>">
                            <h4><?= $productData->productName ?> <?= $stockEmptyMsg ?></h4>
                        </a>

                    </div>
                </div>
                <div class="cart_table_data cart_UnitPriceInner">
                    <div>
                        <p>AED <?= number_format($productOption->productCost, 2) ?></p>
                    </div>
                </div>
                <div class="cart_table_data cart_QtyInner">
                    <div>

                        <?= $record['quantity'] ?>

                    </div>
                </div>
                <div class="cart_table_data cart_SubtotalInner">
                    <div>
                        <h4>AED <?= number_format($productTotal, 2) ?></h4>
                    </div>
                </div>

            </div>
            <?php } ?>
        </div>
    </div>

    <form action="<?= route('confirmPayment') ?>" id="confirmPayment"
        onsubmit="checkPaymentSelected('confirmPayment'); return false;" method="POST">
        <?= csrf_field() ?>

        <div class="cartTotal_detailBox">
            <div class="row">
                <div class="col-md-6">
                    <div class="row">
                        <div class="col-xs-8">
                            <input class="form-control couponcode" placeholder="Coupon Code" id="couponCode"
                                name="couponcode" type="text">
                        </div>
                        <div class="col-xs-4 paddoff">
                            <button type="button" class="btn couponBtn" id="coupon_form">Apply</button>
                            <!-- <button type="submit" class="mobeditcarbtn onlymob">Apply</button> -->
                        </div>
                    </div>
                    <!-- </form> -->
                    <span id="couponMsg"></span>
                </div>
                <div class="col-md-6">
                    <div class="cartTotalInner">
                        <h3>Sub Total <span> AED <?= number_format($subTotal, 2) ?></span></h3>
                        <div id="couponDiscount"></div>
                        <div id="total"></div>
                        <!-- <h3>Total <span id="totalPayable"></span></h3> -->
                        <?php
                                    // if($checkStoreCatIds!=''){

                                    //     $storeCats = explode(',', $checkStoreCatIds);

                                    //     if(in_array($cartData->categoryId, $storeCats)){
                                    //         $expressDelivery = 'Yes';
                                    //         $delivery_charge = 0;
                                    //     }else{
                                    //         $expressDelivery = 'No';
                                    //         $delivery_charge = $delivery_charge;
                                    //     }

                                    // }
                                    // dd($subTotal);
                                    $grandTotal = $subTotal;



                                    if ($userAddress->ship_province == 'Dubai') {

                                        if ($subTotal < $dubai_deliveryamount_limit && $subTotal > 0) { ?>

                        <!-- <h3>Normal Delivery (1-2 Working days) <span id="deliveryCharge"> AED  <?= number_format($dubai_delivery_charge, 2) ?></span></h3> -->

                        <input type="hidden" name="dubaiDeliveryCharge" id="dubaiDeliveryCharge"
                            value="<?= $dubai_delivery_charge ?>">
                        <br>

                        <input type="hidden" name="deliveryChargeAmt" id="deliveryChargeAmt"
                            value="<?= $dubai_delivery_charge ?>" class="deliveryChargeAmt">

                        <div class="form-check">
                            <label class="form-check-label">
                                <input type="radio" class="form-check-input" name="deliveryOption"
                                    value="deliveryChargeDubai" checked
                                    onclick="getDeliveryOptions(this.value, '<?= $grandTotal ?>', '0', '<?= $userAddress->ship_province ?>', '<?= $dubai_delivery_charge ?>');"><span
                                    class="normalCharge">Normal Delivery (1-2 Working days)</span>
                            </label>
                            <span id="deliveryCharge" class="chargeAmt"> AED
                                <?= number_format($dubai_delivery_charge, 2) ?></span>
                        </div>
                        <?php
                                            $grandTotal += $dubai_delivery_charge;
                                        } else { ?>

                        <br>
                        <input type="hidden" name="deliveryChargeAmt" id="deliveryChargeAmt" value="0"
                            class="deliveryChargeAmt">
                        <!-- <h3 class="freeDel"><b>Free Delivery (1-2 Working days)</b> <span> AED  0.00</span></h3> -->
                        <div class="form-check">
                            <label class="form-check-label">
                                <input type="radio" class="form-check-input" name="deliveryOption"
                                    value="deliveryChargeDubai" checked
                                    onclick="getDeliveryOptions(this.value, '<?= $grandTotal ?>', '0', '<?= $userAddress->ship_province ?>', '0');"><span
                                    class="normalCharge">Free Delivery (1-2 Working days)</span>
                            </label>
                            <span id="deliveryCharge" class="chargeAmt"> AED 0.00</span>
                        </div>
                        <?php } ?>

                        <?php } else if ($userAddress->ship_province == 'Sharjah') {
                                        // dd($userAddress->ship_province);
                                        if ($subTotal < $sharjah_deliveryamount_limit && $subTotal > 0) { ?>

                        <!-- <h3>Normal Delivery (1-2 Working days)  <span id="deliveryCharge"> AED  <?= number_format($sharjah_delivery_charge, 2) ?></span></h3> -->

                        <input type="hidden" name="sharjahDeliveryCharge" id="sharjahDeliveryCharge"
                            value="<?= $sharjah_delivery_charge ?>">
                        <br>
                        <input type="hidden" name="deliveryChargeAmt" id="deliveryChargeAmt"
                            value="<?= $sharjah_delivery_charge ?>" class="deliveryChargeAmt">

                        <div class="form-check">
                            <label class="form-check-label">
                                <input type="radio" class="form-check-input" name="deliveryOption"
                                    value="deliveryChargeSharjah" checked
                                    onclick="getDeliveryOptions(this.value, '<?= $grandTotal ?>', '0', '<?= $userAddress->ship_province ?>', '<?= $sharjah_delivery_charge ?>');"><span
                                    class="normalCharge"></span>Normal Delivery (1-2 Working days)</span>
                            </label>
                            <span id="deliveryCharge" class="chargeAmt"> AED
                                <?= number_format($sharjah_delivery_charge, 2) ?></span>
                        </div>

                        <?php
                                            $grandTotal += $sharjah_delivery_charge;
                                        } else { ?>

                        <br>
                        <input type="hidden" name="deliveryChargeAmt" id="deliveryChargeAmt" value="0"
                            class="deliveryChargeAmt">
                        <!-- <h3 class="freeDel"><b>Free Delivery (1-2 Working days)</b> <span> AED  0.00</span></h3> -->
                        <div class="form-check">
                            <label class="form-check-label">
                                <input type="radio" class="form-check-input" name="deliveryOption"
                                    value="deliveryChargeSharjah" checked
                                    onclick="getDeliveryOptions(this.value, '<?= $grandTotal ?>', '0', '<?= $userAddress->ship_province ?>', '0');"><span
                                    class="normalCharge">Free Delivery (1-2 Working days)</span>
                            </label>
                            <span id="deliveryCharge" class="chargeAmt"> AED 0.00</span>
                        </div>

                        <?php } ?>

                        <?php } else if ($userAddress->ship_province == 'Abudhabi/Al ain') {
                                        // dd($userAddress->ship_province);
                                        if ($subTotal < $other_country_deliveryamount_limit && $subTotal > 0) { ?>

                        <!-- western_country_delivery_charge -->
                        <input type="hidden" name="westernCountryDeliveryCharge" id="westernCountryDeliveryCharge"
                            value="<?= $western_country_delivery_charge ?>">

                        <br>
                        <input type="hidden" name="deliveryChargeAmt" id="deliveryChargeAmt"
                            value="<?= $western_country_delivery_charge ?>" class="deliveryChargeAmt">

                        <div class="form-check">
                            <label class="form-check-label">
                                <input type="radio" class="form-check-input" name="deliveryOption"
                                    value="deliveryChargeWesternCountry" checked
                                    onclick="getDeliveryOptions(this.value, '<?= $grandTotal ?>', '0', '<?= $userAddress->ship_province ?>', '<?= $western_country_delivery_charge ?>');"><span
                                    class="normalCharge">Normal Delivery (1-2 Working days)</span>
                            </label>
                            <span id="deliveryCharge" class="chargeAmt"> AED
                                <?= number_format($western_country_delivery_charge, 2) ?></span>
                        </div>
                        <?php
                                            $grandTotal += $western_country_delivery_charge;
                                        } else { ?>

                        <br>
                        <input type="hidden" name="deliveryChargeAmt" id="deliveryChargeAmt" value="0"
                            class="deliveryChargeAmt">
                        <!-- <h3 class="freeDel"><b>Free Delivery (1-2 Working days)</b> <span> AED  0.00</span></h3> -->
                        <div class="form-check">
                            <label class="form-check-label">
                                <input type="radio" class="form-check-input" name="deliveryOption"
                                    value="deliveryChargeWesternCountry" checked
                                    onclick="getDeliveryOptions(this.value, '<?= $grandTotal ?>', '0', '<?= $userAddress->ship_province ?>', '0');"><span
                                    class="normalCharge">Free Delivery (1-2 Working days)</span>
                            </label>
                            <span id="deliveryCharge" class="chargeAmt"> AED 0.00</span>
                        </div>

                        <?php } ?>
                        <?php } else {

                                        if ($subTotal < $other_country_deliveryamount_limit && $subTotal > 0) { ?>

                        <!-- <h3>Normal Delivery (1-2 Working days)  <span id="deliveryCharge"> AED  <?= number_format($other_country_delivery_charge, 2) ?></span></h3> -->
                        <input type="hidden" name="otherCountryDeliveryCharge" id="otherCountryDeliveryCharge"
                            value="<?= $other_country_delivery_charge ?>">

                        <br>
                        <input type="hidden" name="deliveryChargeAmt" id="deliveryChargeAmt"
                            value="<?= $other_country_delivery_charge ?>" class="deliveryChargeAmt">

                        <div class="form-check">
                            <label class="form-check-label">
                                <input type="radio" class="form-check-input" name="deliveryOption"
                                    value="deliveryChargeOtherCountry" checked
                                    onclick="getDeliveryOptions(this.value, '<?= $grandTotal ?>', '0', '<?= $userAddress->ship_province ?>', '<?= $other_country_delivery_charge ?>');"><span
                                    class="normalCharge">Normal Delivery (1-2 Working days)</span>
                            </label>
                            <span id="deliveryCharge" class="chargeAmt"> AED
                                <?= number_format($other_country_delivery_charge, 2) ?></span>
                        </div>
                        <?php
                                            $grandTotal += $other_country_delivery_charge;
                                        } else { ?>

                        <br>
                        <input type="hidden" name="deliveryChargeAmt" id="deliveryChargeAmt" value="0"
                            class="deliveryChargeAmt">
                        <!-- <h3 class="freeDel"><b>Free Delivery (1-2 Working days)</b> <span> AED  0.00</span></h3> -->
                        <div class="form-check">
                            <label class="form-check-label">
                                <input type="radio" class="form-check-input" name="deliveryOption"
                                    value="deliveryChargeOtherCountry" checked
                                    onclick="getDeliveryOptions(this.value, '<?= $grandTotal ?>', '0', '<?= $userAddress->ship_province ?>', '0');"><span
                                    class="normalCharge">Free Delivery (1-2 Working days)</span>
                            </label>
                            <span id="deliveryCharge" class="chargeAmt"> AED 0.00</span>
                        </div>

                        <?php } ?>
                        <?php } ?>

                        <br>
                        <input type="hidden" name="totalItemsCount" id="totalItemsCount" value="<?= $totalItems ?>">

                        <script type="text/javascript">
                        jQuery(document).ready(function() {
                            jQuery('.totalItems').html('<?= $totalItems ?>');
                            jQuery('.mobItemCount').html('<?= $totalItems ?>');
                        });
                        </script>

                        <?php if ($userAddress->ship_province == 'Dubai') { ?>
                        <input type="hidden" name="expressAmt" id="expressAmt" value="">
                        <div class="form-check">
                            <label class="form-check-label">
                                <input type="radio" class="form-check-input" name="deliveryOption" value="express1"
                                    onclick="getDeliveryOptions(this.value, '<?= $subTotal ?>', '<?= $dubai_express_delivery_cost ?>', '<?= $userAddress->ship_province ?>', '0');"><span
                                    class="normalCharge">Express Delivery Within 2-4 HRS –
                                    <?= $dubai_express_delivery_cost ?> AED (Only available within working days & hours,
                                    Saturday-Thursday, 10 AM-6 PM)</span>
                            </label>
                        </div>
                        <?php } ?>

                        <?php if ($userAddress->ship_province == 'Sharjah' || $userAddress->ship_province == 'Abu Dhabi' || $userAddress->ship_province == 'Ajman') { ?>
                        <input type="hidden" name="expressAmt" id="expressAmt" value="">
                        <div class="form-check">
                            <label class="form-check-label">
                                <input type="radio" class="form-check-input" name="deliveryOption" value="express2"
                                    onclick="getDeliveryOptions(this.value, '<?= $subTotal ?>', '<?= $other_express_delivery_cost ?>', '<?= $userAddress->ship_province ?>', '0');"><span
                                    class="normalCharge">Express Delivery Within 3-6 HRS –
                                    <?= $other_express_delivery_cost ?> AED (Only available within working days & hours,
                                    Saturday-Thursday, 10 AM-6 PM)</span>
                            </label>
                        </div>
                        <?php } ?>

                        <br>
                        <div class="form-check">
                            <label class="form-check-label">
                                <input type="radio" class="form-check-input" name="deliveryOption" value="orderPickUp"
                                    onclick="getDeliveryOptions(this.value, '<?= $subTotal ?>', '0', '<?= $userAddress->ship_province ?>', '0');"><span
                                    class="normalCharge">Order And Pick Up ( From Al Quoz, Dubai) We will send you the
                                    collection details when your order is ready (0-1 Days and Should be Collected within
                                    3 Days (Between 10AM to 6PM) Once Order is Ready)</span>
                            </label>
                        </div><br>

                        <?php

                                    $taxAmount = ($grandTotal * 5) / 100;
                                    ?>

                        <h3 id="expressCharge" class="expressDeliveryCharge"></h3>

                        <h3>VAT (5%) <span id="totalTaxAmount"> AED <?= number_format($taxAmount, 2) ?></span></h3>

                        <?php

                                    $grandTotalPlusTax = $grandTotal + $taxAmount;

                                    ?>

                        <h2>Total Payable <span id="grandTotalAmount"> AED
                                <?= number_format($grandTotalPlusTax, 2) ?></span></h2>


                        <?php if ($userAddress->ship_province == 'Dubai') { ?>
                        <?php
                                        if ($subTotal < $dubai_deliveryamount_limit && $subTotal > 0) { ?>

                        <p>(Shop for AED <?= $dubai_deliveryamount_limit ?> or more for FREE delivery)</p>

                        <?php } ?>
                        <?php } elseif ($userAddress->ship_province == 'Sharjah') { ?>
                        <?php
                                        if ($subTotal < $sharjah_deliveryamount_limit && $subTotal > 0) { ?>

                        <p>(Shop for AED <?= $sharjah_deliveryamount_limit ?> or more for FREE delivery)</p>

                        <?php } ?>
                        <?php } else { ?>
                        <?php
                                        if ($subTotal < $other_country_deliveryamount_limit && $subTotal > 0) { ?>

                        <p>(Shop for AED <?= $other_country_deliveryamount_limit ?> or more for FREE delivery)</p>

                        <?php } ?>
                        <?php } ?>
                        <input type="hidden" name="discountVal" id="discountAmt">
                    </div>
                </div>
            </div>
        </div>

        <div class="paymentOptionBox pt_30">
            <h4>Select a payment method you prefer</h4>

            <!-- <form> -->
            <div class="row">
                <div class="p_relative paymentRadioBox mb-1 col-sm-4" id="coddiv">
                    <input class="form-check-input" type="radio" name="paymentType" id="cod" value="COD"
                        onclick="selectPaymentOptions();">
                    <label class="form-check-label" for="cod">
                        <img src="<?= url('/') ?>/images/icons/cash-on-delivery.svg" class="paymentIcon">
                        Cash on Delivery
                    </label>
                </div>
                <div class="p_relative paymentRadioBox mb-1 col-sm-4" id="btdiv">
                    <input class="form-check-input" type="radio" name="paymentType" id="bankTransfer"
                        value="Bank Transfer" onclick="selectPaymentOptions();">
                    <label class="form-check-label" for="bankTransfer">
                        <img src="<?= url('/') ?>/images/icons/transfer.svg" class="paymentIcon">
                        Bank Transfer (Minimum amount 100 AED)
                        <!-- <p class="pay_sub_text">(Make your payment directly into our bank account. Please use your Order Number as the payment reference. Your order will not be shipped until the funds have cleared in our account.)</p> -->
                    </label>
                </div>
                <div class="p_relative paymentRadioBox mb-1 col-sm-4" id="opdiv">
                    <input class="form-check-input" type="radio" name="paymentType" id="onliePayment" value="Online"
                        onclick="selectPaymentOptions();">
                    <label class="form-check-label" for="onliePayment">
                        <img src="<?= url('/') ?>/images/icons/online-payment.svg" class="paymentIcon">
                        Online Payment (Via Credit/Debit Card)
                    </label>
                </div>
            </div>
            <!-- </form> -->

        </div>

        <h3 id="paymentTypeBoxMsg" class="text-center paymentErrorMsg" style="display: none;"></h3>

        <div class="custom-control custom-checkbox mt-3 mb-3">
            <input type="checkbox" class="custom-control-input customCheck" id="customCheck" name="example1"
                onclick="selectTermsAndConditions();">
            <label class="custom-control-label termsChk" for="customCheck">I have read and agree to the website <a
                    href="<?= url('/') ?>/termsandconditions" title="Terms and Conditions" target="_blank"> terms and
                    conditions</a><sup>*</sup></label>
            <h3 id="termsConditionsMsg" class="text-center paymentErrorMsg" style="display: none;"></h3>
        </div>

        <div class="cartFooter d-flex justify-content-center justify-content-sm-between flex-wrap">

            <a href="<?= url('/') ?>/cart" class="btn shopContinueBtn mb-2">View Cart</a>


            <!-- <button class="btn checkout_Btn cartCheckoutBtn mb-2">Proceed to Payment </button> -->

            <!-- <button class="btn checkout_Btn cartCheckoutBtn mb-2">Proceed to Confirm Order </button> -->


            <button class="btn checkout_Btn cartCheckoutBtn mb-2" id="submit_btn">Place order</button>

            <div class="btn checkout_Btn cartCheckoutBtn mb-2" id="process_btn" style="display: none;">Processing...
            </div>
        </div>
    </form>

</div>

<?php } else { ?>
<div class="cartPage_wrapper">
    <h4 class="text-center">Your cart is empty!</h4>
    <button class="shoppingbtn" onclick="window.location.href='<?= url('/') ?>'">
        Continue Shopping <i class="fa fa-shopping-cart"></i>
    </button>
</div>
<?php

            }

    ?>

<?php

    }



    public function confirmPayment(Request $request)
    {

        // dd($request->location);
        // date_default_timezone_set("Asia/Calcutta");
        date_default_timezone_set('Asia/Dubai');
        $dateTime  = date('Y-m-d H:i:s', time());
        $session_id = session()->get('session_id');

        // $free_deliveryamount_limit = Config::get('constants.FREE_DELIVERYAMOUNT_LIMIT');
        // $delivery_charge = Config::get('constants.DELIVERY_CHARGE');

        $business_offer_percent = Config::get('constants.BUSINESS_OFFER_PERCENT');
        $offer_percent = Config::get('constants.OFFER_PERCENT');
        $checkStoreCatIds = Config::get('constants.STORE_CATEGORY_ID');

        $dubai_deliveryamount_limit = Config::get('constants.DUBAI_DELIVERY_LIMIT');
        $dubai_delivery_charge = Config::get('constants.DUBAI_DELIVERY_CHARGE');

        $sharjah_deliveryamount_limit = Config::get('constants.SHARJAH_DELIVERY_LIMIT');
        $sharjah_delivery_charge = Config::get('constants.SHARJAH_DELIVERY_CHARGE');

        $other_country_deliveryamount_limit = Config::get('constants.OTHER_COUNTRY_DELIVERY_LIMIT');
        $other_country_delivery_charge = Config::get('constants.OTHER_COUNTRY_DELIVERY_CHARGE');

        $western_country_delivery_charge = Config::get('constants.WESTERN_COUNTRY_DELIVERY_CHARGE');

        $dubai_express_delivery_cost = Config::get('constants.DUBAI_EXPRESS_DELIVERY_COST');
        $other_express_delivery_cost = Config::get('constants.OTHER_EXPRESS_DELIVERY_COST');

        if ($session_id == '') {
            return view('home');
        }

        if ($request->deliveryOption == 'express1') {
            $expressDeliveryCost = $dubai_express_delivery_cost;
            $express = 'Yes';
            $orderAndPickUp = 'No';
            $deliveryOption = 'ExpressDelivery';
        } elseif ($request->deliveryOption == 'express2') {
            $expressDeliveryCost = $other_express_delivery_cost;
            $express = 'Yes';
            $orderAndPickUp = 'No';
            $deliveryOption = 'ExpressDelivery';
        } elseif ($request->deliveryOption == 'orderPickUp') {
            $expressDeliveryCost = '0';
            $express = 'No';
            $orderAndPickUp = 'Yes';
            $deliveryOption = 'OrderPickup';
        } else {
            $expressDeliveryCost = '0';
            $express = 'No';
            $orderAndPickUp = 'No';
        }

        $cartData = Cart::select(['id', 'categoryId'])->where('cartId', $session_id)->orderBy('id', 'DESC')->first();

        $regId = Auth::user()->id;
        $mobileNumber = Auth::user()->mobileNumber;
        $email = Auth::user()->email;



        $userData = User::select(['id', 'mobileNumber'])->where('email', $email)->first();

        $useraddresses = UserAddresses::where('userId', $regId)->first();

        $registereduser = RegisteredUser::where('id', $regId)->first();

        $registereduser->companyName = $useraddresses->company_name;
        $registereduser->location = $useraddresses->location;
        $registereduser->trn = $useraddresses->trn;
        $registereduser->address = $useraddresses->address;
        $registereduser->country = $useraddresses->country;
        $registereduser->province = $useraddresses->province;
        $registereduser->ship_fullName = $useraddresses->ship_fullName;
        $registereduser->ship_country_code = $useraddresses->ship_country_code;
        $registereduser->ship_mobileNumber = $useraddresses->ship_mobileNumber;
        $registereduser->ship_location = $useraddresses->ship_location;
        $registereduser->ship_country = $useraddresses->ship_country;
        $registereduser->ship_province = $useraddresses->ship_province;
        $registereduser->ship_address = $useraddresses->ship_address;

        $update = $registereduser->save();

        if (count($userData) == 0) {



            $user = new User();

            $user->fullName = $useraddresses->fullName;
            $user->email = $useraddresses->email;
            $user->location = $useraddresses->location;
            $user->companyName = $useraddresses->company_name;
            // $user->pincode = $useraddresses->pincode;
            $user->trn = $useraddresses->trn;
            $user->country_code = $useraddresses->country_code;
            $user->mobileNumber = $useraddresses->mobileNumber;
            $user->password = str_random(8);
            $user->address = $useraddresses->address;
            $user->country = $useraddresses->country;
            $user->province = $useraddresses->province;
            $user->dateTime = $dateTime;
            $user->save();


            $userId = $user->id;
        } else {

            $userId = $userData->id;
        }



        // dd($userId);
        if ($userId) {


            $invoiceNoOrderDetails = OrderDetails::select(['invoiceNo'])->whereNotNull('invoiceNo')->orderBy('id', 'DESC')->first();

            if (count($invoiceNoOrderDetails) > 0 && $invoiceNoOrderDetails->invoiceNo != '') {
                $oldinvoice = substr($invoiceNoOrderDetails->invoiceNo, 1);
                $invoiceNo = $oldinvoice + 1;
                $newinvoiceNo = 'B' . $invoiceNo;
            } else if (count($invoiceNoOrderDetails) > 0 && $invoiceNoOrderDetails->invoiceNo == '') {
                $newinvoiceNo = 'B6464';
            } else {

                $newinvoiceNo = 'B6464';
            }

            $orderdetails = new OrderDetails();

            $orderdetails->userId = $userId;
            $orderdetails->regId = $regId;
            $orderdetails->customerId = $regId;
            $orderdetails->cartId = $session_id;
            $orderdetails->invoiceNo = $newinvoiceNo;
            $orderdetails->addressId = $useraddresses->id;
            $orderdetails->country_code = $useraddresses->country_code;
            $orderdetails->mobileNumber = $useraddresses->mobileNumber;
            $orderdetails->fullName = $useraddresses->fullName;
            $orderdetails->trn = $useraddresses->trn;
            $orderdetails->companyName = $useraddresses->company_name;
            $orderdetails->email = $useraddresses->email;
            $orderdetails->location = $useraddresses->location;
            $orderdetails->address = $useraddresses->address;
            $orderdetails->country = $useraddresses->country;
            $orderdetails->province = $useraddresses->province;
            $orderdetails->ship_fullName = $useraddresses->ship_fullName;
            $orderdetails->ship_country_code = $useraddresses->ship_country_code;
            $orderdetails->ship_mobileNumber = $useraddresses->ship_mobileNumber;
            $orderdetails->ship_location = $useraddresses->ship_location;
            $orderdetails->ship_address = $useraddresses->ship_address;
            $orderdetails->ship_country = $useraddresses->ship_country;
            $orderdetails->ship_province = $useraddresses->ship_province;
            $orderdetails->note = $useraddresses->customer_note;
            $orderdetails->dateTime = $dateTime;
            $orderdetails->expressDeliveryCost = $expressDeliveryCost;
            $orderdetails->express = $express;
            $orderdetails->orderAndPickUp = $orderAndPickUp;
            // $orderdetails->paymentType = 'Online';
            $orderdetails->paymentType = $request->paymentType;
            // $orderdetails->onlinePaymentStatus = 'Pending';

            $res = $orderdetails->save();

            $orderId = $orderdetails->id;

            if ($res) {

                $records = Cart::select(['id', 'productId', 'productOptionId', 'quantity', 'timeslotVal'])->where('cartId', $session_id)->get();


                $totalItems = 0;

                if (count($records) > 0) {

                    $subTotal = 0;
                    $grandtotal = 0;
                    $productTotal = 0;
                    $totalItems = 0;

                    foreach ($records as $record) {

                        $productData = Products::select(['id', 'productName', 'categoryId'])->where('id', $record['productId'])->first();

                        $productOption = productOptions::select(['id', 'productWeight', 'productUnit', 'productCost', 'productStock', 'orderCount', 'productOffer', 'productDistributerPrice'])->where('id', $record['productOptionId'])->first();

                        if ((isset($productOption->productOffer)) && ($productOption->productOffer != '') && ($productOption->productOffer > 0) && (count($productOption->productOffer) > 0)) {
                            $offerPrice = round(($productOption->productOffer * $productOption->productCost) / 100);
                            if ($offerPrice < 1) {
                                $offerPrice = 1;
                            }
                            $productOption->productCost = ($productOption->productCost - $offerPrice);
                        }

                        $unitPrice = $productOption->productCost;
                        $productTotalPrice = $record['quantity'] * $productOption->productCost;

                        $orderedItems = new OrderItems();

                        $orderedItems->orderId = $orderId;
                        $orderedItems->productId = $record['productId'];
                        $orderedItems->productName = $productData->productName;
                        $orderedItems->productOptionId = $record['productOptionId'];
                        $orderedItems->productWeight = $productOption->productWeight;
                        $orderedItems->productUnit = $productOption->productUnit;
                        $orderedItems->unitPrice = $unitPrice;
                        $orderedItems->quantity = $record['quantity'];
                        $orderedItems->productTotalPrice = $productTotalPrice;
                        $orderedItems->productCost = $productOption->productCost;
                        $orderedItems->productDistributerPrice = $productOption->productDistributerPrice;

                        $orderItemInsert = $orderedItems->save();

                        if ($orderItemInsert) {

                            $orderCount = $productOption->orderCount + $record['quantity'];

                            // Stock Update ----

                            $productStock = 0;

                            $currentStock = $productOption->productStock;
                            $updatedStock = $currentStock - $record['quantity'];

                            if ($updatedStock >= 0) {
                                $productStock = $updatedStock;
                            } else {
                            }

                            //-----------------

                            $productOptionsUpdate = productOptions::where('id', $productOption->id)->first();

                            $productOptionsUpdate->orderCount = $orderCount;

                            $updatePO = $productOptionsUpdate->save();

                            $productTotal = $record['quantity'] * $productOption->productCost;
                            $totalItems += $record['quantity'];
                            $subTotal += $productTotal;
                        }
                    }



                    if ($orderItemInsert) {

                        $grandTotal = $subTotal;

                        if (session()->get('couponAvailable') == 'Yes') {
                            $couponDiscount = session()->get('couponAmount');
                        } else {
                            $couponDiscount = 0;
                        }

                        $grandTotal -= $couponDiscount;

                        $deliveryCost = 0;
                        $offerTotal = 0;

                        if ($useraddresses->ship_province == 'Dubai') {

                            if ($subTotal < $dubai_deliveryamount_limit && $subTotal > 0) {
                                if ($request->deliveryOption == 'orderPickUp') {
                                    $deliveryCost = 0;
                                    // $grandTotal+=$dubai_delivery_charge;
                                } else if ($request->deliveryOption == 'express1') {
                                    $deliveryCost = 0;
                                } else if ($request->deliveryOption == 'express2') {
                                    $deliveryCost = 0;
                                } else {
                                    $deliveryCost = $dubai_delivery_charge;
                                    $grandTotal += $dubai_delivery_charge;
                                }
                            }
                        } elseif ($useraddresses->ship_province == 'Sharjah') {

                            if ($subTotal < $sharjah_deliveryamount_limit && $subTotal > 0) {
                                if ($request->deliveryOption == 'orderPickUp') {
                                    $deliveryCost = 0;
                                } else if ($request->deliveryOption == 'express1') {
                                    $deliveryCost = 0;
                                } else if ($request->deliveryOption == 'express2') {
                                    $deliveryCost = 0;
                                } else {
                                    $deliveryCost = $sharjah_delivery_charge;
                                    $grandTotal += $sharjah_delivery_charge;
                                }
                            }
                        } elseif ($useraddresses->ship_province == 'Abudhabi/Al ain') {

                            if ($subTotal < $other_country_deliveryamount_limit && $subTotal > 0) {

                                if ($request->deliveryOption == 'orderPickUp') {
                                    $deliveryCost = 0;
                                } else if ($request->deliveryOption == 'express1') {
                                    $deliveryCost = 0;
                                } else if ($request->deliveryOption == 'express2') {
                                    $deliveryCost = 0;
                                } else {

                                    $deliveryCost = $western_country_delivery_charge;
                                    $grandTotal += $western_country_delivery_charge;
                                }
                            }
                        } else {

                            if ($subTotal < $other_country_deliveryamount_limit && $subTotal > 0) {

                                if ($request->deliveryOption == 'orderPickUp') {
                                    $deliveryCost = 0;
                                } else if ($request->deliveryOption == 'express1') {
                                    $deliveryCost = 0;
                                } else if ($request->deliveryOption == 'express2') {
                                    $deliveryCost = 0;
                                } else {

                                    $deliveryCost = $other_country_delivery_charge;
                                    $grandTotal += $other_country_delivery_charge;
                                }
                            }
                        }

                        if ($request->deliveryOption == 'orderPickUp') {
                            $deliveryOption = 'orderPickUp';
                        } else if ($request->deliveryOption == 'express1') {
                            $deliveryOption = 'ExpressDelivery';
                        } else if ($request->deliveryOption == 'express2') {
                            $deliveryOption = 'ExpressDelivery';
                        } else {

                            if ($deliveryCost > 0) {
                                $deliveryOption = 'NormalDelivery';
                            } else {
                                $deliveryOption = 'FreeDelivery';
                            }
                        }



                        $grandTotal += $expressDeliveryCost;

                        // $grandTotal-= $couponDiscount;

                        $tax = ($grandTotal * 5) / 100;

                        $grandTotal += $tax;
                        // dd($grandTotal);
                        $ORDERNUMBER = $orderId + 1000;
                        if (session()->get('businessUserOffer') == 'Yes') {
                            $offerTotal = round(($business_offer_percent / 100) * $subTotal);
                            if ($offerTotal < 1) {
                                $offerTotal = 1;
                            }

                            session()->forget('BUSINESS_OFFER_PERCENT');
                            session()->forget('businessUserOfferAmount');
                        }
                        if (session()->get('offerAvailable') == 'Yes') {
                            $offerTotal = round(($offer_percent / 100) * $subTotal);
                            if ($offerTotal < 1) {
                                $offerTotal = 1;
                            }
                            // unset($_SESSION['offerAvailable']);
                            // unset($_SESSION['offerAmount']);
                            session()->forget('offerAvailable');
                            session()->forget('offerAmount');
                        }

                        // if(session()->get('couponAvailable')!='Yes'){
                        //     $couponDiscount=0;
                        // }

                        if (session()->get('couponAvailable') == 'Yes') {
                            $couponDiscount = session()->get('couponAmount');
                            $regId = Auth::user()->id;

                            $couponusers = new CouponUsers();

                            $couponusers->regId = $regId;
                            $couponusers->orderId = $orderId;
                            $couponusers->couponId = session()->get('couponId');
                            $couponusers->couponCode = session()->get('couponCode');
                            $couponusers->dateTime = $dateTime;
                            $couponusers->userMobileNumber = $request->mobileNumber;

                            $couponUsersInsert = $couponusers->save();


                            session()->forget('couponAvailable');
                            session()->forget('couponAmount');
                            session()->forget('couponCode');
                            session()->forget('couponId');
                        }


                        $orderdetailsupdate = OrderDetails::where('id', $orderId)->first();

                        $orderdetailsupdate->invoiceNumber = $ORDERNUMBER;
                        $orderdetailsupdate->subTotal = $subTotal;
                        $orderdetailsupdate->deliveryCost = $deliveryCost;
                        $orderdetailsupdate->deliveryOption = $deliveryOption;
                        $orderdetailsupdate->totalAmount = $grandTotal;
                        $orderdetailsupdate->tax = $tax;
                        $orderdetailsupdate->offerAmt = $offerTotal;
                        $orderdetailsupdate->couponDiscount = $couponDiscount;

                        $rs = $orderdetailsupdate->save();
                    }
                }
            }
        }


        if ($rs) {

            $useradddata = UserAddresses::where('id', $request->addressId)->first();

            if ($request->paymentType == 'Online') {

                // $merchantTxnId = date('ymdhms').'_'.$ORDERNUMBER;

                if ($offerTotal > 0) {
                    $grandTotal = $grandTotal - $offerTotal;
                }

                // if($couponDiscount > 0){
                //   $grandTotal = $grandTotal - $couponDiscount;
                // }



                if ($grandTotal < 0) {
                    $grandTotal = 0;
                }


                Session::put('orderinvoice', $ORDERNUMBER);

                $curl = curl_init();
                curl_setopt_array($curl, array(
                    CURLOPT_URL => "https://foloosi.com/api/v1/api/initialize-setup",
                    CURLOPT_RETURNTRANSFER => true,
                    CURLOPT_ENCODING => "",
                    CURLOPT_MAXREDIRS => 10,
                    CURLOPT_TIMEOUT => 30,
                    CURLOPT_HTTP_VERSION => CURL_HTTP_VERSION_1_1,
                    CURLOPT_CUSTOMREQUEST => "POST",
                    CURLOPT_POSTFIELDS => "transaction_amount=$grandTotal&currency=AED&customer_address=$useraddresses->address&customer_city=$useraddresses->location&billing_country=ARE&billing_state=$useraddresses->province&customer_name=$useraddresses->fullName&customer_email=$useraddresses->email&customer_mobile=$useraddresses->mobileNumber",
                    CURLOPT_HTTPHEADER => array(
                        "content-type: application/x-www-form-urlencoded",
                        "merchant_key: live_\$2y\$10\$9mco80RAz2Vb2BQDrfNXAOQbSVN5Vs6OFqwrcB.xbPoN1UHQkA7-K"
                    ),
                ));
                $response = curl_exec($curl);
                $err = curl_error($curl);
                curl_close($curl);
                // print_r($response);
                if ($err) {
                    echo "cURL Error #:" . $err;
                } else {
                    $responseData = json_decode($response, true);
                    // dd($responseData);
                    $reference_token = $responseData['data']['reference_token'];
                }

                // $updated = OrderDetails::where('invoiceNumber', $ORDERNUMBER)->update(['order_ref' => $reference_token, 'paymentType' => 'Online']);

    ?>
<!DOCTYPE html>
<html>

<head>
    <title>Payment</title>
    <meta charset="utf-8" />
    <meta name="csrf-token" content="<?= csrf_token() ?>">
</head>

<body>
    <script src="https://duboxx.ae/js/jquery-3.2.1.min.js"></script>
    <script type="text/javascript" src="https://www.foloosi.com/js/foloosipay.v2.js"></script>

    <script type="text/javascript">
    $.ajaxSetup({

        headers: {

            'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')

        }

    });

    var reference_token = "<?= $reference_token; ?>";
    var orderId = "<?= $orderId; ?>";
    var options = {
        "reference_token": reference_token,
        "merchant_key": "live_$2y$10$9mco80RAz2Vb2BQDrfNXAOQbSVN5Vs6OFqwrcB.xbPoN1UHQkA7-K"

    }
    var fp1 = new Foloosipay(options);
    fp1.open();
    foloosiHandler(response, function(e) {
        if (e.data.status == 'success') {

            $.ajax({
                type: "POST",
                url: "<?= route('saveTransactionNo') ?>",
                data: {
                    "orderId": orderId,
                    "transaction_no": e.data.data.transaction_no,
                    "_token": "<?= csrf_token() ?>"
                },

                success: function(msg) {

                    window.location.href = "https://duboxx.ae/paymentsuccess";
                }
            });

        }
        if (e.data.status == 'error') {
            // console.log(e.data)
            // window.location.href="https://duboxx.ae/failure";
            $.ajax({
                type: "POST",
                url: "<?= route('saveTransactionNo') ?>",
                data: {
                    "orderId": orderId,
                    "transaction_no": e.data.data.transaction_no,
                    "_token": "<?= csrf_token() ?>"
                },

                success: function(msg) {

                    window.location.href = "https://duboxx.ae/failure";
                }
            });
        }
        if (e.data.status == 'closed') {
            // console.log(e.data)
            window.location.href = "https://duboxx.ae/proceedToPayment";
        }
    });
    </script>

</body>

</html>

<?php

            } elseif ($request->paymentType == 'COD' || $request->paymentType == 'Bank Transfer') {


                self::emailNotifications($orderId);
                self::emailNotificationsAdmin($orderId);

                if (isset($session_id) && $session_id != '') {
                    $delete = DB::table('cart')->where('cartId', $session_id)->delete();
                }


                session()->forget('session_id');

                // $_SESSION['orderinvoice'] = $ORDERNUMBER;

                // session()->put('orderinvoice') = $ORDERNUMBER;
                Session::put('orderinvoice', $ORDERNUMBER);
                // return $ORDERNUMBER;
                return redirect()->route('thankyou');
            }
        }
    }


    public  function paymentsuccess()
    {

        $orderId = session()->get('orderinvoice');


        $session_id = session()->get('session_id');
        $onlinePaymentStatus = 'Complete';

        if ($session_id != '') {
            $delete = DB::table('cart')->where('cartId', $session_id)->delete();
        }

        session()->forget('session_id');

        $updated = OrderDetails::where('invoiceNumber', $orderId)->update(['onlinePaymentStatus' => $onlinePaymentStatus]);



        $orderDetails = OrderDetails::select(['id'])->where('invoiceNumber', $orderId)->first();



        self::emailNotifications($orderDetails->id);
        self::emailNotificationsAdmin($orderDetails->id);


        return redirect()->route('thankyou');
    }

    public function foloosipayment()
    {

        $orderinvoice = session()->get('orderinvoice');

        $orderDetails = OrderDetails::select(['id', 'userId', 'invoiceNumber', 'invoiceNo', 'fullName', 'mobileNumber', 'email', 'address', 'location', 'pincode', 'landmark', 'note', 'subTotal', 'deliveryCost', 'totalAmount', 'dateTime', 'offerAmt', 'couponDiscount', 'giftOrder', 'expressDeliveryCost', 'deliverytimeSlot', 'deliveryDisplaySlot', 'merchantTxnId', 'paymentType', 'country', 'province', 'ship_fullName', 'ship_location', 'ship_country', 'ship_province', 'ship_address', 'companyName', 'ship_country_code', 'trn', 'country_code', 'ship_mobileNumber', 'tax', 'paymentType'])->where('invoiceNumber', $orderinvoice)->first();

        return view('foloosipayment', compact('orderDetails'));
    }

    public function saveTransactionNo(Request $request)
    {

        $updateTransactionNo = DB::table('order_details')->where('id', $request->orderId)->update(array('order_ref' => $request->transaction_no));
    }
}
