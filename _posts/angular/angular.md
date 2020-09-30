---
title: Шпаргалка по Angular
description: 
---

## Создание проекта, компонента...

```bash
npm install -g @angular/cli
ng new project-name
ng generate component component-name
ng add @angular/material
ng serve
```

## Привязки, циклы

```html

<input type="text" [(ngModel)]="currentProduct" >
<button (click)="onAddProduct()">Add product</button>
<div *ngFor="let product of products">
  {{product}}
</div>
```

## Прокидывание свойств внутрь компонента

```html
<app-product [productName]="product" (productClicked)="onProductCliced(product)"></app-product>
```

```typescript
import {Component, EventEmitter, Input, OnInit, Output} from '@angular/core';

  @Input() productName: string;
  @Output() productClicked = new EventEmitter();

```

## Angular material

### Install

```bash
ng add @angular/material
```

### button

```html
<button mat-icon-button (click)="onAddProduct()">
  <mat-icon>
    home
  </mat-icon>
```

### валидация формы

```html
<section>
  <form fxLayout="column"  fxLayoutAlign="center center" #f="ngForm" (ngSubmit)="onSubmit(f)">
    <mat-form-field>
      <input type="email"
             matInput
             placeholder="email"
             ngModel name="email"
             email
             required
             #emailInput="ngModel"
      >

      <mat-error *ngIf="emailInput.hasError('required')">the field must not be empty</mat-error>
      <mat-error *ngIf="!emailInput.hasError('required')">email is invalid</mat-error>
    </mat-form-field>

    <mat-form-field hintLabel="Should be 6 character long at least">
      <input type="password"
             matInput
             placeholder="password"
             ngModel name="password"
             required
             minlength="6"
             #pwInput="ngModel"
      >
      <mat-hint align="end">{{pwInput.value?.length}}/6</mat-hint>
    </mat-form-field>

    <button type="submit" mat-raised-button color="primary">Submit</button>

  </form>
</section>
```

