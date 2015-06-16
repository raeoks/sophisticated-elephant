---
layout: post
title: "HMAC-Based API Authentication"
date: 2015-04-07 19:48:55 +0530
comments: true
categories: [hmac, api, rails, ruby, restful]
---

Username and Password based authentication is for humans. It is insecure for
communication between applications. To establish a secure communication between
REST client and service we prefer HMAC method for authentication. And to keep
things simple,  we put in place an example to authenticate client requests.
We'll hand roll a REST client using ruby net http library. Connect that client
to E4 payment gateway and accept telecheck payments. E4 exposes a legacy REST
services. Which doesn't encrypt transaction and response code is always
HTTP OK - 200. We should always encrypt any sensitive data sent to any REST
service and use appropriate HTTP codes in responses.

When we consider security for REST based APIs, we often go with HTTPS. HTTPS
creates a secure channel over an insecure network. This ensures reasonable
protection from eavesdroppers and man-in-the-middle attacks, provided that
adequate cipher suites are used and that the server certificate is verified
and trusted. HTTPS piggybacks HTTP entirely on top of TLS, the entirety of
the underlying HTTP protocol can be encrypted. However when we need an
additional level of security, HTTPS doesn't cut out well. We need an
alternative, for instance we might need to track usage of our API for each
customer and need to know exactly who's making the calls.

A common way to solve this problem is by signing the message based on a shared
secret between the client and service. This signature is called HMAC
(Hash-based Message Authentication Code), with which we create a message
authentication code (MAC) for a request based on a secret key that we've shared.

Client calculates the HMAC key based of HTTP-Method, Content-SHA1, Content-Type,
Data Header and Path. Client then sends that signed request to service. Service
on the other hand normalizes the request in the same manner as the client did,
and calculates the HMAC value. If the HMAC from the client matches the
calculated HMAC from the server. Surety of message integrity is guaranteed and
client can be easily identified.

{% highlight ruby %}
require 'spec_helper'
 
module Telecheck
  describe Payment do
    context 'with a valid customer and bank' do
      it 'should perform an approved transactions and responds with success' do
        customer_name       = 'John Doe'
        customer_email      = 'john.doe@example.com'
        bank_account_number = '123456789'
        bank_routing_number = '123123123'
        dollar_amount       = Money.new(1337, 'USD')
        payment             = Payment.new(customer_name, customer_email, bank_account_number,
                                          bank_routing_number, dollar_amount)
        payment.purchase!
        expect(payment).to be_success
      end
    end
 
    context 'with a invalid customer and bank' do
      it 'should perform an disapproved transactions and responds with failure' do
        customer_name       = 'John Doe'
        customer_email      = 'john.doe@example.com'
        bank_account_number = '123456'
        bank_routing_number = '123789'
        dollar_amount       = Money.new(1337, 'USD')
        payment             = Payment.new(customer_name, customer_email, bank_account_number,
                                          bank_routing_number, dollar_amount)
        payment.purchase!
        expect(payment).not_to be_success
      end
    end
  end
end
{% endhighlight %}

{% highlight ruby %}
# performs telecheck payments transactions
 
module Telecheck
  class Payment
    attr_reader :ctr, :errors
 
    def initialize(customer_name, customer_email, bank_account_number,
                       bank_routing_number, dollar_amount)
      @customer_name       = customer_name
      @customer_email      = customer_email
      @bank_account_number = bank_account_number
      @bank_routing_number = bank_routing_number
      @dollar_amount       = dollar_amount
      @timestamp           = Time.now.getgm.iso8601
      @errors              = []
      @ctr                 = ''
    end
 
    def purchase!
      send_request
    end
 
    def success?
      @success
    end
 
    private
 
    KEY_ID       = ENV['TELECHECK_API_ACCESS_KEY_ID']
    KEY          = ENV['TELECHECK_API_ACCESS_HMAC_KEY']
    HOST         = ENV['TELECHECK_HOST']
    END_POINT    = '/transaction/v14'
    CONTENT_TYPE = 'text/xml; charset=UTF-8'
    CRYPTO       = 'sha1'
    METHOD       = 'POST'
    NEW_LINE     = "\n"
 
 
    def content
      "<?xml version=\"1.0\" encoding=\"UTF-8\"?> #{Gyoku.xml({
      'Transaction' => {
        'ExactID'           => ENV['TELECHECK_GATEWAY_ID'],
        'PASSWORD'          => ENV['TELECHECK_GATEWAY_PASSWORD'],
        'Transaction_Type'  => '00',
        'CustomerName'      => @customer_name,
        'ClientEmail'       => @customer_email,
        'BankAccountNumber' => @bank_account_number,
        'BankRoutingNumber' => @bank_routing_number,
        'DollarAmount'      => @dollar_amount.cents
      }
    })}".encode('ascii')
    end
 
    def headers
      {
        'Content-Type'        => CONTENT_TYPE,
        'X-GGe4-Content-SHA1' => content_digest,
        'X-GGe4-Date'         => @timestamp,
        'Authorization'       => "GGE4_API #{KEY_ID}:#{hmac}"
      }
    end
 
    def content_digest
      Digest::SHA1.hexdigest(content)
    end
 
    def hmac
      Base64.encode64(OpenSSL::HMAC.digest(CRYPTO, KEY, 
                          authorization_data))[0..-2]
    end
 
    def authorization_data
      "#{METHOD}#{NEW_LINE}#{CONTENT_TYPE}#{NEW_LINE}#{content_digest}" + 
          "#{NEW_LINE}#{@timestamp}#{NEW_LINE}#{END_POINT}".encode('ascii')
    end
 
    def uri
      URI.parse(HOST)
    end
 
    def send_request
      http = Net::HTTP.new(uri.host, uri.port)
      http.use_ssl = true
      http.verify_mode = OpenSSL::SSL::VERIFY_NONE
      response = http.post(END_POINT, content, headers)
      begin
        result = Hash.from_xml(response.body)
      rescue REXML::ParseException
        result = {}
        @errors += ["#{response.code}: #{response.body}"]
      end
      if result['TransactionResult'] && 
            result['TransactionResult']['Transaction_Approved'] == 'true'
        @ctr     = result['TransactionResult']['CTR']
        @success = true
      else
        @success = false
      end
    end
  end
end
{% endhighlight %}

##Conclusion

Different REST services uses different ways to calculate HMAC. Some uses new lines 
(prefer using "\n" instead of '\n') while other uses commas or any other character 
for separating parameters. Read the API documentation carefully because if you miss 
out on one small thing like parameters separator. This will result in wrong HMAC 
calculation and API request won't authorize.
