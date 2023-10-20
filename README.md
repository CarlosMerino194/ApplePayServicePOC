# ApplePayServicePOC

This is a POC of an implementation for Apple Pay service:

```swift
import Foundation
import PassKit
import Combine

struct OrderInformation {
    var orderID: String
    var customerName: String
    var address: String
    var phoneNumber: String
    var postalAddress: String
    var emailAddress: String
}

protocol ApplePayServiceType {
    var result: PassthroughSubject<OrderInformation, PKPaymentError> { get }
    func startPaymentProcess(label: String, amount: Double)
}

class ApplePayService: NSObject, ApplePayServiceType {
    let supportedNetworks: [PKPaymentNetwork] = [
        .amex,
        .discover,
        .masterCard,
        .visa
    ]
    
    var result: PassthroughSubject<OrderInformation, PKPaymentError> = .init()
    
    func applePayStatus() -> (canMakePayments: Bool, canSetupCards: Bool) {
        return (PKPaymentAuthorizationController.canMakePayments(),
                PKPaymentAuthorizationController.canMakePayments(usingNetworks: supportedNetworks))
    }
    
    func startPaymentProcess(label: String, amount: Double) {
        let value = String(format: "%.2f", amount)
        let item = PKPaymentSummaryItem(label: label, amount: NSDecimalNumber(string: value))
        
        let paymentRequest = PKPaymentRequest()
        paymentRequest.paymentSummaryItems = [item]
        paymentRequest.merchantIdentifier = ""
        paymentRequest.countryCode = "US"
        paymentRequest.currencyCode = "USD"
        paymentRequest.merchantCapabilities = .capability3DS
        paymentRequest.requiredBillingContactFields = [.emailAddress, .phoneNumber, .name, .postalAddress]
        paymentRequest.supportedNetworks = supportedNetworks
        
        lauchPaymentController(with: paymentRequest)
    }
    
    private func lauchPaymentController(with paymentRequest: PKPaymentRequest) {
        let paymentController = PKPaymentAuthorizationController(paymentRequest: paymentRequest)
        paymentController.delegate = self
        paymentController.present() {[weak self] completion in
            guard !completion else {
                return
            }
            
            self?.result.send(completion: .failure(PKPaymentError(.unknownError)))
        }
    }
    
    private func getBillingInformation(_ info: PKContact?) {
        guard let contact = info else {
            result.send(completion: .failure(PKPaymentError(.billingContactInvalidError)))
            return
        }
        
        let addressFormatter = CNPostalAddressFormatter()
        let address = addressFormatter.string(from: contact.postalAddress ?? .init())
        
        let orderInformation = OrderInformation(orderID: UUID().uuidString,
                                                customerName: "\(contact.name?.givenName ?? "") \(contact.name?.familyName ?? "")",
                                                address: address,
                                                phoneNumber: contact.phoneNumber?.stringValue ?? "",
                                                postalAddress: contact.postalAddress?.postalCode ?? "",
                                                emailAddress: contact.emailAddress ?? "")
        result.send(orderInformation)
    }
}

extension ApplePayService: PKPaymentAuthorizationControllerDelegate {
    func paymentAuthorizationControllerDidFinish(_ controller: PKPaymentAuthorizationController) {
        controller.dismiss()
    }
    
    func paymentAuthorizationController(
        _ controller: PKPaymentAuthorizationController,
        didAuthorizePayment payment: PKPayment,
        handler completion: @escaping (PKPaymentAuthorizationResult) -> Void
    ) {
        getBillingInformation(payment.billingContact)
        completion(PKPaymentAuthorizationResult(status: .success, errors: nil))
    }
}
```
