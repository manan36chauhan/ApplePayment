# React native Apple Payment integration
[Manzsoftech](https://medium.com/@manzsoftech)

## Need to create cerrtification and development merchent id in apple developer account then create certificate and download also create p12 file for specific payment gateway like stripe/braintree/bambora etc.

**required to create 4 file , ApplePaymentModule.m, ApplePaymentModule.swift, useApplePay.ts (typescript) screen.js**

    //create file ApplePymenModule.swift


    import Foundation
    import PassKit
    import React

    struct PaymentRequestParams: Codable {
      let merchantIdentifier: String
      let supportedNetworks: [String]
      let countryCode: String
      let currencyCode: String
      let label: String
      let amount: String
    }

    @objc(ApplePayModule)
    class ApplePayModule: NSObject {
      private var paymentResolver: RCTPromiseResolveBlock?
      private var paymentRejecter: RCTPromiseRejectBlock?
      private var completionHandler: ((PKPaymentAuthorizationResult) -> Void)?
  
    @objc
    static func requiresMainQueueSetup() -> Bool {
      return true
    }
  
    @objc
    func canMakePayments(_ resolver: RCTPromiseResolveBlock, rejecter: RCTPromiseRejectBlock) {
      resolver(PKPaymentAuthorizationController.canMakePayments())
    }
  
    @objc
    func requestPayment(_ paramsJSON: NSDictionary, resolver: @escaping RCTPromiseResolveBlock, rejecter: @escaping RCTPromiseRejectBlock) {
      guard let jsonData = try? JSONSerialization.data(withJSONObject: paramsJSON, options: []),
            let params = try? JSONDecoder().decode(PaymentRequestParams.self, from: jsonData) else {
        rejecter("E_INVALID_PARAMS", "Invalid payment request parameters", nil)
        return
      }
  
    let paymentRequest = PKPaymentRequest()
    paymentRequest.merchantIdentifier = params.merchantIdentifier
    paymentRequest.supportedNetworks = mapSupportedNetworks(params.supportedNetworks)
    paymentRequest.merchantCapabilities = .capability3DS
    paymentRequest.countryCode = params.countryCode
    paymentRequest.currencyCode = params.currencyCode
    paymentRequest.paymentSummaryItems = [
      PKPaymentSummaryItem(label: params.label, amount: NSDecimalNumber(string: params.amount))
    ]

    let paymentAuthorizationController = PKPaymentAuthorizationController(paymentRequest: paymentRequest)
    paymentAuthorizationController.delegate = self
    paymentAuthorizationController.present { (presented: Bool) in
      if !presented {
        rejecter("E_PAYMENT_ERROR", "Unable to present Apple Pay authorization.", nil)
      } else {
        self.paymentResolver = resolver
        self.paymentRejecter = rejecter
      }
    }
    }
  
    @objc
    func completePayment(_ success: Bool) {
      guard let completionHandler = self.completionHandler else {
        return
      }
      let status: PKPaymentAuthorizationStatus = success ? .success : .failure
      let result = PKPaymentAuthorizationResult(status: status, errors: nil)
      completionHandler(result)
      self.completionHandler = nil
    }
  
    private func mapSupportedNetworks(_ networks: [String]) -> [PKPaymentNetwork] {
      return networks.compactMap { network in
        switch network.lowercased() {
        case "visa":
          return .visa
        case "mastercard":
          return .masterCard
        case "mada":
          return .mada
        case "amex":
          return .amex
        case "discover":
          return .discover
        case "jcb":
          return .JCB
        case "maestro":
          return .maestro
        case "electron":
          return .electron
        case "vpay":
          return .vPay
        default:
          return nil
        }
      }
    }
      }
  
    extension ApplePayModule: PKPaymentAuthorizationControllerDelegate {
      func paymentAuthorizationController(_ controller: PKPaymentAuthorizationController, didAuthorizePayment payment: PKPayment, handler completion: @escaping (PKPaymentAuthorizationResult) -> Void) {
        // Store the completion handler
        self.completionHandler = completion
  
    // Handle the payment authorization
    if let resolver = self.paymentResolver {
      do {
        let paymentData = try JSONSerialization.jsonObject(with: payment.token.paymentData, options: []) as? [String: Any] ?? [:]
        resolver(["status": "success", "paymentData": paymentData])
      } catch {
        if let rejecter = self.paymentRejecter {
          rejecter("APPLE_PAY_PAYMENT_REJECTED", "Payment was rejected", error)
        }
        completion(PKPaymentAuthorizationResult(status: .failure, errors: nil))
        return
      }
      self.paymentResolver = nil
      self.paymentRejecter = nil
    } else {
      // Resolver is nil, reject the payment
      if let rejecter = self.paymentRejecter {
        rejecter("APPLE_PAY_PAYMENT_REJECTED", "Payment was rejected", nil)
        self.paymentResolver = nil
        self.paymentRejecter = nil
      }
      completion(PKPaymentAuthorizationResult(status: .failure, errors: nil))
    }
    }
  
    func paymentAuthorizationControllerDidFinish(_ controller: PKPaymentAuthorizationController) {
      controller.dismiss {
        // Handle the dismissal of the payment sheet
        if let rejecter = self.paymentRejecter {
          rejecter("APPLE_PAY_PAYMENT_CANCELLED", "Payment was cancelled", nil)
          self.paymentResolver = nil
          self.paymentRejecter = nil
        }
      }
    }
    } 
  
### create ApplePayModule.m module file

    //objective C language create module file ApplePayModule.m in xcode
    #import <React/RCTBridgeModule.h>
    
    @interface RCT_EXTERN_MODULE(ApplePayModule, NSObject)
    
    RCT_EXTERN_METHOD(canMakePayments:(RCTPromiseResolveBlock)resolver rejecter:(RCTPromiseRejectBlock)rejecter)
    RCT_EXTERN_METHOD(requestPayment:(NSDictionary *)paramsJSON resolver:(RCTPromiseResolveBlock)resolver rejecter:(RCTPromiseRejectBlock)rejecter)
    RCT_EXTERN_METHOD(completePayment:(BOOL)success)
    
    @end
  ### create Hook file
  
    //useApplePay.ts
  
  
    import {Alert, NativeModules, Platform} from 'react-native';
  
    export enum Country {
      AE = 'AE', // United Arab Emirates
      BH = 'BH', // Bahrain
      KW = 'KW', // Kuwait
      OM = 'OM', // Oman
      QA = 'QA', // Qatar
      SA = 'SA', // Saudi Arabia
      US = 'US', // United States
      GB = 'GB', // United Kingdom
      IN = 'IN', // India
      CA = 'CA', // Canada
      AU = 'AU', // Australia
      DE = 'DE', // Germany
      FR = 'FR', // France
      SG = 'SG', // Singapore
    }
  
    export enum Currency {
      AED = 'AED', // UAE Dirham
      BHD = 'BHD', // Bahraini Dinar
      KWD = 'KWD', // Kuwaiti Dinar
      OMR = 'OMR', // Omani Rial
      QAR = 'QAR', // Qatari Riyal
      SAR = 'SAR', // Saudi Riyal
      GBP = 'GBP', // British Pound
      USD = 'USD', // US Dollar
      INR = 'INR', // Indian Rupee
      CAD = 'CAD', // Canadian Dollar
      AUD = 'AUD', // Australian Dollar
      EUR = 'EUR', // Euro
      SGD = 'SGD', // Singapore Dollar
    }
  
    export enum APPLE_PAY_SUPPORTED_NETWORKS {
      AMEX = 'amex', // American Express
      BANCONTACT = 'bancontact', // Bancontact
      CARTES_BANCAIRES = 'cartesBancaires', // Cartes Bancaires
      CHINA_UNION_PAY = 'chinaUnionPay', // China Union Pay
      DANKORT = 'dankort', // Dankort
      DISCOVER = 'discover', // Discover
      EFTPOS = 'eftpos', // EFTPOS
      ELECTRON = 'electron', // Visa Electron
      ELO = 'elo', // ELO
      GIROCARD = 'girocard', // Girocard
      INTERAC = 'interac', // Interac
      JCB = 'jcb', // JCB
      MADA = 'mada', // MADA
      MAESTRO = 'maestro', // Maestro
      MASTERCARD = 'masterCard', // MasterCard
      MIR = 'mir', // MIR
      PRIVATE_LABEL = 'privateLabel', // Private Label Cards
      VISA = 'visa', // Visa
      VPAY = 'vPay', // VPay
      UNION_PAY = 'unionPay', // Union Pay (Global)
      TROY = 'troy', // Troy (Turkey)
      PULSE = 'pulse', // Pulse
      STAR = 'star', // Star Network
      NYCE = 'nyce', // NYCE
      ACCEL = 'accel', // Accel
      ZIP = 'zip', // Zip (Buy Now Pay Later)
      BARCLAYCARD = 'barclaycard', // Barclaycard
    }
  
    export interface ApplePayToken {
      token: string;
      ephemeralPublicKey: string;
      publicKeyHash: string;
      signature: string;
      transactionId: string;
      version: string;
    }
    
    interface PaymentDetailsType {
      country: Country;
      currency: Currency;
      amount: number;
      label: string;
    }
    
    interface UseApplePayProps {
      handlePayment: (values: any, paymentResponse: any) => Promise<any>;
      isCreatSub?: boolean;
    }
  
    interface OnApplePayProps {
      paymentValues: PaymentDetailsType;
      paymentRequestBody?: any;
      paymentKey?: string;
      merchantId: string;
    }
    
    interface ApplePayRequestData {
      merchantIdentifier: string;
      supportedNetworks: APPLE_PAY_SUPPORTED_NETWORKS[];
      countryCode: string;
      currencyCode: string;
      label: string;
      amount: string;
    }
  
    export enum APPLE_PAY_RESPONSE_CODES {
      APPLE_PAY_PAYMENT_REJECTED = 'APPLE_PAY_PAYMENT_REJECTED',
      APPLE_PAY_PAYMENT_CANCELLED = 'APPLE_PAY_PAYMENT_CANCELLED',
    }
    
    const {ApplePayModule} = NativeModules;
    
    const supportedNetworks = [
      APPLE_PAY_SUPPORTED_NETWORKS.VISA,
      APPLE_PAY_SUPPORTED_NETWORKS.MASTERCARD,
      APPLE_PAY_SUPPORTED_NETWORKS.AMEX,
      APPLE_PAY_SUPPORTED_NETWORKS.MADA,
    ];
    
    const useApplePay = ({handlePayment, isCreatSub}: UseApplePayProps) => {
      const canMakePayments = async () => {
        try {
          if (Platform.OS === 'ios') {
            return await ApplePayModule.canMakePayments();
          }
          return false;
        } catch (error) {
          return false;
        }
      };
  
    const requestPayment = async (data: ApplePayRequestData) => {
      if (Platform.OS === 'ios') {
        return await ApplePayModule.requestPayment(data);
      }
      throw new Error('Apple Pay is not supported on this platform.');
    };
  
    const handleApplePay = async (data: ApplePayRequestData) => {
      const isAvailable = await canMakePayments();
  
    if (!isAvailable) {
      throw new Error('Apple Pay is not available on this device');
    }

    try {
      const paymentResponse = await requestPayment(data);
      if (paymentResponse.status === 'success') {
        return paymentResponse.paymentData;
      }
      throw new Error('Payment failed');
    } catch (error) {
      throw error;
    }
    };
  
    const alertError = () => {
      Alert.alert('somethingWentWrong');
    };
  
    const onApplePay = async ({
      paymentRequestBody = {},
      paymentValues,
      paymentKey = isCreatSub ? 'token' : 'source',
      merchantId,
    }: OnApplePayProps) => {
 
    
    try {
      if (!paymentValues.currency || !paymentValues.amount) {
        console.log("--------------------------------------------",paymentValues);
        alertError();
      } else {
        const merchantIdentifier = merchantId;
        const paymentInternalInfo = {
          merchantIdentifier,
          country: paymentValues.country,
          currencyCode: paymentValues.currency,
        };

        const isPaymentPossible = await canMakePayments();

        if (isPaymentPossible) {
          const paymentResponse = await handleApplePay({
            merchantIdentifier,
            supportedNetworks,
            countryCode: paymentValues.country,
            currencyCode: paymentValues.currency,
            label: 'Motmaina',
            amount: paymentValues.amount.toString(),
          });

          console.log(JSON.stringify(paymentResponse, null, 2));

          if (paymentResponse) {
            const {version, data: token, header, signature} = paymentResponse;
            const {transactionId, publicKeyHash, ephemeralPublicKey} = header;
            const values = {
              version,
              token,
              transactionId,
              publicKeyHash,
              ephemeralPublicKey,
              signature,
            } as ApplePayToken;

            try {
              await handlePayment(
                {
                  [paymentKey]: values,
                  ...paymentRequestBody,
                },
                paymentResponse,
              );
              await ApplePayModule.completePayment(true);
            } catch (error) {
              throw new Error(
                `handle payment error ${{...paymentInternalInfo, ...values}}`,
              );
            }
          } else {
            throw new Error('Empty Payment Response');
          }
        } else {
          throw new Error(
            `Payment is not possible ${{...paymentInternalInfo}}`,
          );
        }
      }
    } catch (error: any) {
      console.log(error);
      
      if (error.code !== APPLE_PAY_RESPONSE_CODES.APPLE_PAY_PAYMENT_CANCELLED) {
        ApplePayModule.completePayment(false);
        alertError();
      }
    }
    };
    return {
      onApplePay,
    };
    };
      export default useApplePay;

## then create button and implement apple pay

    //ApplePayDemo.js
    
    import useApplePay, {
      Country,
      Currency,
    } from "../../ComponentReuse/useApplePay";
    
    
      const ApplePayDemo = () => {
  
    const { onApplePay } = useApplePay({
      handlePayment: async (values, paymentResponse) => {
        try {
          // code for API call to backend for payment processing go here
          console.log("Processing payment with values:", values, paymentResponse);
          const response = await fetch(
            "https://your-backend-api.com/process-payment",
            {
              method: "POST",
              headers: { "Content-Type": "application/json" },
              body: JSON.stringify(values),
            }
          );

        if (!response.ok) {
          throw new Error("Payment processing failed");
        }

        return response.json();
      } catch (error) {
        console.error("Error processing payment:", error);
        throw new Error("Failed to process payment");
      }
    },
    isCreatSub: false,
    });
  
    const initiateApplePay = async () => {
      try {
        await onApplePay({
          paymentValues: {
            country: Country.CA,
            currency: Currency.CAD,
            amount: 75.0,
            label: "Test Product",
          },
          merchantId: "merchant.com.pizzaiolo.srx", // need to replace with newly created merchant id
        });
      } catch (error) {
        console.error("Payment failed:", error.message);
        Alert.alert("Error", error.message || "Payment failed");
      }
    };
  
  
  
    return (
      <View style={{ marginTop: 150}}>
        <Button title="Apple Pay" onPress={initiateApplePay} />
      </View>
    );
      };
  
      export default ApplePayButton;
