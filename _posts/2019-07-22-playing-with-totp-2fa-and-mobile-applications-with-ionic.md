---
layout: post
title: "Playing with TOTP (2FA) and mobile applications with ionic"
date: "2019-07-22"
tags: 
  - "js"
  - "python"
  - "technology"
  - "2fa"
  - "flask"
  - "ionic"
  - "totp"
coverImage: "118669.png"
---

Today I want to play with Two Factor Authentication. When we speak about 2FA, TOTP come to our mind. There're a lot of TOTP clients, for example Google Authenticator.

My idea with this prototype is to build one Mobile application (with ionic) and validate one totp token in a server (in this case a Python/Flask application). The token will be generated with a standard TOTP client. Let's start

The sever will be a simple Flask server to handle routes. One route (GET /) will generate one QR code to allow us to configure or TOTP client. I'm using the library pyotp to handle totp operations.

\[sourcecode language="python"\] from flask import Flask, jsonify, abort, render\_template, request import os from dotenv import load\_dotenv from functools import wraps import pyotp from flask\_qrcode import QRcode

current\_dir = os.path.dirname(os.path.abspath(\_\_file\_\_)) load\_dotenv(dotenv\_path="{}/.env".format(current\_dir))

totp = pyotp.TOTP(os.getenv('TOTP\_BASE32\_SECRET'))

app = Flask(\_\_name\_\_) QRcode(app)

def verify(key): return totp.verify(key)

def authorize(f): @wraps(f) def decorated\_function(\*args, \*\*kws): if not 'Authorization' in request.headers: abort(401)

data = request.headers\['Authorization'\] token = str.replace(str(data), 'Bearer ', '')

if token != os.getenv('BEARER'): abort(401)

return f(\*args, \*\*kws)

return decorated\_function

@app.route('/') def index(): return render\_template('index.html', totp=pyotp.totp.TOTP(os.getenv('TOTP\_BASE32\_SECRET')).provisioning\_uri("gonzalo123.com", issuer\_name="TOTP Example"))

@app.route('/check/<key>', methods=\['GET'\]) @authorize def alert(key): status = verify(key) return jsonify({'status': status})

if \_\_name\_\_ == "\_\_main\_\_": app.run(host='0.0.0.0') \[/sourcecode\]

I'll use an standard TOTP client to generate the tokens but with pyotp we can easily create a client also

\[sourcecode language="python"\] import pyotp import time import os from dotenv import load\_dotenv import logging

logging.basicConfig(level=logging.INFO)

current\_dir = os.path.dirname(os.path.abspath(\_\_file\_\_)) load\_dotenv(dotenv\_path="{}/.env".format(current\_dir))

totp = pyotp.TOTP(os.getenv('TOTP\_BASE32\_SECRET'))

mem = None while True: now = totp.now() if mem != now: logging.info(now) mem = now time.sleep(1) \[/sourcecode\]

And finally the mobile application. It's a simple ionic application. That's the view:

\[sourcecode language="html"\] <ion-header> <ion-toolbar> <ion-title> TOTP Validation demo </ion-title> </ion-toolbar> </ion-header>

<ion-content> <div class="ion-padding"> <ion-item> <ion-label position="stacked">totp</ion-label> <ion-input placeholder="Enter value" \[(ngModel)\]="totp"></ion-input> </ion-item> <ion-button fill="solid" color="secondary" (click)="validate()" \[disabled\]="!totp"> Validate <ion-icon slot="end" name="help-circle-outline"></ion-icon> </ion-button> </div> </ion-content> \[/sourcecode\]

The controller:

\[sourcecode language="js"\] import { Component } from '@angular/core' import { ApiService } from '../sercices/api.service' import { ToastController } from '@ionic/angular'

@Component({ selector: 'app-home', templateUrl: 'home.page.html', styleUrls: \['home.page.scss'\] }) export class HomePage { public totp

constructor (private api: ApiService, public toastController: ToastController) {}

validate () { this.api.get('/check/' + this.totp).then(data => this.alert(data.status)) }

async alert (status) { const toast = await this.toastController.create({ message: status ? 'OK' : 'Not valid code', duration: 2000, color: status ? 'primary' : 'danger', }) toast.present() } } \[/sourcecode\]

I've also put a simple security system. In a real life application we'll need something better, but here I've got a Auth Bearer harcoded and I send it en every http request. To do it I've created a simple api service

\[sourcecode language="js"\] import { Injectable } from '@angular/core' import { isDevMode } from '@angular/core' import { HttpClient, HttpHeaders, HttpParams } from '@angular/common/http' import { CONF } from './conf'

@Injectable({ providedIn: 'root' }) export class ApiService {

private isDev: boolean = isDevMode() private apiUrl: string

constructor (private http: HttpClient) { this.apiUrl = this.isDev ? CONF.API\_DEV : CONF.API\_PROD }

public get (uri: string, params?: Object): Promise<any> { return new Promise((resolve, reject) => { this.http.get(this.apiUrl + uri, { headers: ApiService.getHeaders(), params: ApiService.getParams(params) }).subscribe( res => {this.handleHttpNext(res), resolve(res)}, err => {this.handleHttpError(err), reject(err)}, () => this.handleHttpComplete() ) }) }

private static getHeaders (): HttpHeaders {

const headers = { 'Content-Type': 'application/json' }

headers\['Authorization'\] = 'Bearer ' + CONF.bearer

return new HttpHeaders(headers) }

private static getParams (params?: Object): HttpParams { let Params = new HttpParams() for (const key in params) { if (params.hasOwnProperty(key)) { Params = Params.set(key, params\[key\]) } }

return Params }

private handleHttpError (err) { console.log('HTTP Error', err) }

private handleHttpNext (res) { console.log('HTTP response', res) }

private handleHttpComplete () { console.log('HTTP request completed.') } } \[/sourcecode\]

And that's all. Here one video with a working example of the prototype:

\[embed\]https://www.youtube.com/watch?v=bgljQu0RVNs\[/embed\]

Source code [here](https://github.com/gonzalo123/totp)
