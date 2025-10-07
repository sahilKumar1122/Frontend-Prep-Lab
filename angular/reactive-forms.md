# Angular Reactive Forms

## Table of Contents
- [Reactive Forms Fundamentals](#reactive-forms-fundamentals)
- [Form Validation](#form-validation)
- [Dynamic Forms](#dynamic-forms)
- [Form Arrays](#form-arrays)
- [Custom Validators](#custom-validators)
- [Form Performance](#form-performance)

---

## Reactive Forms Fundamentals

### Question: Explain Reactive Forms in Angular and when to use them over Template-driven forms.

**Answer:**

**Reactive Forms** provide a model-driven approach to handling form inputs with explicit, immutable data flow. They're built around **Observable streams** and offer more power, flexibility, and testability than Template-driven forms.

**Reactive vs Template-Driven:**

| Feature | Reactive Forms | Template-driven Forms |
|---------|---------------|----------------------|
| Setup | Explicit in component | Implicit in template |
| Data model | Structured, immutable | Unstructured, mutable |
| Data flow | Synchronous | Asynchronous |
| Validation | Functions | Directives |
| Testing | Easy | Difficult |
| Complexity | Better for complex | Better for simple |

**Complete Reactive Form Example:**

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { 
  FormBuilder, 
  FormGroup, 
  FormControl, 
  Validators,
  AbstractControl 
} from '@angular/forms';
import { Subject, debounceTime, distinctUntilChanged, takeUntil } from 'rxjs';

@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <!-- Basic input with validation -->
      <div class="form-group">
        <label for="firstName">First Name</label>
        <input 
          id="firstName"
          type="text" 
          formControlName="firstName"
          class="form-control"
          [class.is-invalid]="isFieldInvalid('firstName')"
        />
        <div *ngIf="isFieldInvalid('firstName')" class="error">
          <span *ngIf="userForm.get('firstName')?.hasError('required')">
            First name is required
          </span>
          <span *ngIf="userForm.get('firstName')?.hasError('minlength')">
            Minimum 2 characters required
          </span>
        </div>
      </div>

      <!-- Email with async validation -->
      <div class="form-group">
        <label for="email">Email</label>
        <input 
          id="email"
          type="email" 
          formControlName="email"
          class="form-control"
        />
        <div *ngIf="isFieldInvalid('email')" class="error">
          <span *ngIf="userForm.get('email')?.hasError('required')">
            Email is required
          </span>
          <span *ngIf="userForm.get('email')?.hasError('email')">
            Invalid email format
          </span>
          <span *ngIf="userForm.get('email')?.hasError('emailTaken')">
            Email already exists
          </span>
        </div>
        <span *ngIf="userForm.get('email')?.pending" class="validating">
          Checking email availability...
        </span>
      </div>

      <!-- Nested form group -->
      <div formGroupName="address" class="nested-group">
        <h3>Address</h3>
        
        <div class="form-group">
          <label for="street">Street</label>
          <input 
            id="street"
            type="text" 
            formControlName="street"
            class="form-control"
          />
        </div>

        <div class="form-group">
          <label for="city">City</label>
          <input 
            id="city"
            type="text" 
            formControlName="city"
            class="form-control"
          />
        </div>

        <div class="form-group">
          <label for="zipCode">Zip Code</label>
          <input 
            id="zipCode"
            type="text" 
            formControlName="zipCode"
            class="form-control"
          />
        </div>
      </div>

      <!-- Password with confirmation -->
      <div class="form-group">
        <label for="password">Password</label>
        <input 
          id="password"
          type="password" 
          formControlName="password"
          class="form-control"
        />
      </div>

      <div class="form-group">
        <label for="confirmPassword">Confirm Password</label>
        <input 
          id="confirmPassword"
          type="password" 
          formControlName="confirmPassword"
          class="form-control"
        />
        <div *ngIf="userForm.hasError('passwordMismatch') && 
                    userForm.get('confirmPassword')?.touched" 
             class="error">
          Passwords do not match
        </div>
      </div>

      <!-- Form actions -->
      <div class="form-actions">
        <button 
          type="submit" 
          [disabled]="userForm.invalid || userForm.pending"
          class="btn btn-primary"
        >
          {{ userForm.pending ? 'Validating...' : 'Submit' }}
        </button>
        <button 
          type="button" 
          (click)="resetForm()"
          class="btn btn-secondary"
        >
          Reset
        </button>
      </div>

      <!-- Debug info (remove in production) -->
      <div class="debug-info" *ngIf="isDevelopment">
        <h4>Form Status</h4>
        <pre>{{ userForm.value | json }}</pre>
        <p>Valid: {{ userForm.valid }}</p>
        <p>Touched: {{ userForm.touched }}</p>
        <p>Dirty: {{ userForm.dirty }}</p>
      </div>
    </form>
  `,
  styleUrls: ['./user-form.component.scss']
})
export class UserFormComponent implements OnInit, OnDestroy {
  userForm!: FormGroup;
  isDevelopment = !environment.production;
  private destroy$ = new Subject<void>();

  constructor(
    private fb: FormBuilder,
    private userService: UserService
  ) {}

  ngOnInit(): void {
    this.initializeForm();
    this.setupValueChanges();
  }

  private initializeForm(): void {
    this.userForm = this.fb.group({
      firstName: ['', [
        Validators.required,
        Validators.minLength(2),
        Validators.maxLength(50)
      ]],
      lastName: ['', [
        Validators.required,
        Validators.minLength(2)
      ]],
      email: ['', 
        [Validators.required, Validators.email],
        [this.emailExistsValidator.bind(this)]  // Async validator
      ],
      address: this.fb.group({
        street: [''],
        city: ['', Validators.required],
        zipCode: ['', [
          Validators.required,
          Validators.pattern(/^\d{5}$/)
        ]]
      }),
      password: ['', [
        Validators.required,
        Validators.minLength(8),
        this.passwordStrengthValidator
      ]],
      confirmPassword: ['', Validators.required]
    }, {
      validators: this.passwordMatchValidator  // Form-level validator
    });
  }

  private setupValueChanges(): void {
    // Listen to specific field changes
    this.userForm.get('email')?.valueChanges
      .pipe(
        debounceTime(300),
        distinctUntilChanged(),
        takeUntil(this.destroy$)
      )
      .subscribe(email => {
        console.log('Email changed:', email);
        // Could trigger side effects here
      });

    // Listen to entire form changes
    this.userForm.valueChanges
      .pipe(takeUntil(this.destroy$))
      .subscribe(formValue => {
        // Auto-save to localStorage
        this.autoSaveForm(formValue);
      });

    // Listen to status changes
    this.userForm.statusChanges
      .pipe(takeUntil(this.destroy$))
      .subscribe(status => {
        console.log('Form status:', status);
      });
  }

  // Custom async validator
  private emailExistsValidator(control: AbstractControl) {
    return this.userService.checkEmailExists(control.value)
      .pipe(
        debounceTime(500),
        map(exists => exists ? { emailTaken: true } : null),
        catchError(() => of(null))
      );
  }

  // Custom sync validator (form-level)
  private passwordMatchValidator(group: AbstractControl) {
    const password = group.get('password')?.value;
    const confirmPassword = group.get('confirmPassword')?.value;
    
    return password === confirmPassword ? null : { passwordMismatch: true };
  }

  // Custom validator function
  private passwordStrengthValidator(control: AbstractControl) {
    const value = control.value;
    
    if (!value) return null;
    
    const hasNumber = /[0-9]/.test(value);
    const hasUpper = /[A-Z]/.test(value);
    const hasLower = /[a-z]/.test(value);
    const hasSpecial = /[!@#$%^&*]/.test(value);
    
    const valid = hasNumber && hasUpper && hasLower && hasSpecial;
    
    return valid ? null : { weakPassword: true };
  }

  isFieldInvalid(fieldName: string): boolean {
    const field = this.userForm.get(fieldName);
    return !!(field?.invalid && (field?.dirty || field?.touched));
  }

  onSubmit(): void {
    if (this.userForm.valid) {
      console.log('Form submitted:', this.userForm.value);
      
      // Get only changed values
      const changedValues = this.getChangedValues();
      console.log('Changed values:', changedValues);
      
      this.userService.saveUser(this.userForm.value)
        .pipe(takeUntil(this.destroy$))
        .subscribe({
          next: (response) => {
            console.log('User saved', response);
            this.userForm.markAsPristine();
          },
          error: (error) => {
            console.error('Save failed', error);
            this.handleServerErrors(error);
          }
        });
    } else {
      // Mark all fields as touched to show validation errors
      this.markFormGroupTouched(this.userForm);
    }
  }

  resetForm(): void {
    this.userForm.reset();
  }

  private markFormGroupTouched(formGroup: FormGroup): void {
    Object.keys(formGroup.controls).forEach(key => {
      const control = formGroup.get(key);
      control?.markAsTouched();
      
      if (control instanceof FormGroup) {
        this.markFormGroupTouched(control);
      }
    });
  }

  private getChangedValues(): any {
    const changedValues: any = {};
    
    Object.keys(this.userForm.controls).forEach(key => {
      const control = this.userForm.get(key);
      if (control?.dirty) {
        changedValues[key] = control.value;
      }
    });
    
    return changedValues;
  }

  private handleServerErrors(error: any): void {
    if (error.errors) {
      Object.keys(error.errors).forEach(key => {
        const control = this.userForm.get(key);
        if (control) {
          control.setErrors({ serverError: error.errors[key] });
        }
      });
    }
  }

  private autoSaveForm(formValue: any): void {
    localStorage.setItem('userFormDraft', JSON.stringify(formValue));
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Programmatic Form Control:**

```typescript
// Dynamically add/remove controls
addPhoneNumber(): void {
  this.userForm.addControl('phone', new FormControl('', Validators.required));
}

removePhoneNumber(): void {
  this.userForm.removeControl('phone');
}

// Enable/disable controls
disableEmail(): void {
  this.userForm.get('email')?.disable();
}

enableEmail(): void {
  this.userForm.get('email')?.enable();
}

// Set/patch values
populateForm(userData: User): void {
  // patchValue - partial update
  this.userForm.patchValue({
    firstName: userData.firstName,
    email: userData.email
  });
  
  // setValue - must provide all values
  this.userForm.setValue({
    firstName: userData.firstName,
    lastName: userData.lastName,
    email: userData.email,
    address: {
      street: userData.address.street,
      city: userData.address.city,
      zipCode: userData.address.zipCode
    },
    password: '',
    confirmPassword: ''
  });
}
```

**Pro Tip:** Most developers don't leverage the power of **valueChanges** and **statusChanges** observables. These are incredibly useful for implementing features like auto-save, real-time validation feedback, dependent fields, and analytics. Combine with RxJS operators like `debounceTime`, `distinctUntilChanged`, and `switchMap` for sophisticated form behaviors. Always remember to unsubscribe using `takeUntil` to prevent memory leaks.

---

## Form Validation

### Question: How do you implement complex validation logic in Angular Reactive Forms?

**Answer:**

Angular provides **built-in validators** but real-world applications often require **custom validation logic**. Understanding how to create reusable, composable validators is essential for senior roles.

**Complete Validation Example:**

```typescript
// validators.ts - Reusable Custom Validators
import { AbstractControl, ValidationErrors, ValidatorFn, AsyncValidatorFn } from '@angular/forms';
import { Observable, of, timer } from 'rxjs';
import { map, switchMap, catchError } from 'rxjs/operators';

export class CustomValidators {
  
  // 1. Simple Custom Validator
  static noWhitespace(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      const isWhitespace = (control.value || '').trim().length === 0;
      return isWhitespace ? { whitespace: true } : null;
    };
  }
  
  // 2. Parameterized Validator
  static minAge(age: number): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      
      const birthDate = new Date(control.value);
      const today = new Date();
      const userAge = today.getFullYear() - birthDate.getFullYear();
      
      return userAge >= age ? null : { minAge: { required: age, actual: userAge } };
    };
  }
  
  // 3. Cross-field Validator (FormGroup level)
  static passwordMatch(passwordField: string, confirmField: string): ValidatorFn {
    return (group: AbstractControl): ValidationErrors | null => {
      const password = group.get(passwordField)?.value;
      const confirm = group.get(confirmField)?.value;
      
      return password === confirm ? null : { passwordMismatch: true };
    };
  }
  
  // 4. Conditional Validator
  static conditionalValidator(
    predicate: () => boolean,
    validator: ValidatorFn
  ): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!predicate()) {
        return null;
      }
      return validator(control);
    };
  }
  
  // 5. Composite Validator
  static strongPassword(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      const value = control.value;
      
      if (!value) return null;
      
      const errors: any = {};
      
      if (!/[A-Z]/.test(value)) {
        errors.missingUpperCase = true;
      }
      
      if (!/[a-z]/.test(value)) {
        errors.missingLowerCase = true;
      }
      
      if (!/[0-9]/.test(value)) {
        errors.missingNumber = true;
      }
      
      if (!/[!@#$%^&*(),.?":{}|<>]/.test(value)) {
        errors.missingSpecialChar = true;
      }
      
      if (value.length < 8) {
        errors.minLength = true;
      }
      
      return Object.keys(errors).length > 0 ? errors : null;
    };
  }
  
  // 6. Async Validator with debounce
  static usernameAvailable(userService: UserService): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) {
        return of(null);
      }
      
      return timer(500).pipe(  // Debounce 500ms
        switchMap(() => userService.checkUsername(control.value)),
        map(isTaken => isTaken ? { usernameTaken: true } : null),
        catchError(() => of(null))
      );
    };
  }
  
  // 7. Pattern-based Validators
  static phoneNumber(countryCode: string = 'US'): ValidatorFn {
    const patterns: { [key: string]: RegExp } = {
      US: /^\+?1?\d{10}$/,
      UK: /^\+?44\d{10}$/,
      IN: /^\+?91\d{10}$/
    };
    
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      
      const pattern = patterns[countryCode];
      const valid = pattern.test(control.value);
      
      return valid ? null : { invalidPhone: { country: countryCode } };
    };
  }
  
  // 8. File Upload Validator
  static fileSize(maxSizeInMB: number): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      const file = control.value as File;
      
      if (!file) return null;
      
      const maxSizeInBytes = maxSizeInMB * 1024 * 1024;
      
      return file.size > maxSizeInBytes 
        ? { fileSize: { max: maxSizeInMB, actual: (file.size / 1024 / 1024).toFixed(2) } }
        : null;
    };
  }
  
  static fileType(allowedTypes: string[]): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      const file = control.value as File;
      
      if (!file) return null;
      
      const fileType = file.type;
      const valid = allowedTypes.includes(fileType);
      
      return valid ? null : { fileType: { allowed: allowedTypes, actual: fileType } };
    };
  }
  
  // 9. Credit Card Validator (Luhn Algorithm)
  static creditCard(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      
      const cardNumber = control.value.replace(/\s/g, '');
      
      if (!/^\d+$/.test(cardNumber)) {
        return { invalidCreditCard: true };
      }
      
      let sum = 0;
      let isEven = false;
      
      for (let i = cardNumber.length - 1; i >= 0; i--) {
        let digit = parseInt(cardNumber[i]);
        
        if (isEven) {
          digit *= 2;
          if (digit > 9) digit -= 9;
        }
        
        sum += digit;
        isEven = !isEven;
      }
      
      return sum % 10 === 0 ? null : { invalidCreditCard: true };
    };
  }
  
  // 10. URL Validator
  static url(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      
      try {
        new URL(control.value);
        return null;
      } catch {
        return { invalidUrl: true };
      }
    };
  }
}
```

**Using Custom Validators:**

```typescript
@Component({
  selector: 'app-advanced-form',
  template: `...`
})
export class AdvancedFormComponent implements OnInit {
  form!: FormGroup;
  
  constructor(
    private fb: FormBuilder,
    private userService: UserService
  ) {}
  
  ngOnInit(): void {
    this.form = this.fb.group({
      // Multiple validators
      username: ['', 
        [
          Validators.required,
          Validators.minLength(3),
          CustomValidators.noWhitespace()
        ],
        [CustomValidators.usernameAvailable(this.userService)]  // Async
      ],
      
      // Parameterized validator
      birthDate: ['', [
        Validators.required,
        CustomValidators.minAge(18)
      ]],
      
      // Pattern validator
      phone: ['', [
        Validators.required,
        CustomValidators.phoneNumber('US')
      ]],
      
      // Strong password
      password: ['', [
        Validators.required,
        CustomValidators.strongPassword()
      ]],
      
      confirmPassword: ['', Validators.required],
      
      // File upload
      avatar: ['', [
        CustomValidators.fileSize(2),  // 2MB max
        CustomValidators.fileType(['image/jpeg', 'image/png'])
      ]],
      
      // Credit card
      cardNumber: ['', [
        Validators.required,
        CustomValidators.creditCard()
      ]],
      
      // URL
      website: ['', [
        CustomValidators.url()
      ]],
      
      // Conditional validation
      companyName: ['', [
        CustomValidators.conditionalValidator(
          () => this.form?.get('isCompany')?.value === true,
          Validators.required
        )
      ]],
      
      isCompany: [false]
    }, {
      validators: [
        // Form-level validators
        CustomValidators.passwordMatch('password', 'confirmPassword')
      ]
    });
    
    // Dynamic validation based on other fields
    this.form.get('isCompany')?.valueChanges.subscribe(isCompany => {
      const companyName = this.form.get('companyName');
      
      if (isCompany) {
        companyName?.setValidators([Validators.required]);
      } else {
        companyName?.clearValidators();
      }
      
      companyName?.updateValueAndValidity();
    });
  }
  
  // Display validation errors
  getErrorMessage(controlName: string): string {
    const control = this.form.get(controlName);
    
    if (!control || !control.errors || !control.touched) {
      return '';
    }
    
    const errors = control.errors;
    
    if (errors['required']) return 'This field is required';
    if (errors['minlength']) {
      return `Minimum length is ${errors['minlength'].requiredLength}`;
    }
    if (errors['maxlength']) {
      return `Maximum length is ${errors['maxlength'].requiredLength}`;
    }
    if (errors['email']) return 'Invalid email format';
    if (errors['whitespace']) return 'Cannot be only whitespace';
    if (errors['minAge']) {
      return `Minimum age is ${errors['minAge'].required}`;
    }
    if (errors['usernameTaken']) return 'Username already taken';
    if (errors['invalidPhone']) return 'Invalid phone number';
    if (errors['missingUpperCase']) return 'Must contain uppercase letter';
    if (errors['missingLowerCase']) return 'Must contain lowercase letter';
    if (errors['missingNumber']) return 'Must contain number';
    if (errors['missingSpecialChar']) return 'Must contain special character';
    if (errors['fileSize']) {
      return `File size must be less than ${errors['fileSize'].max}MB`;
    }
    if (errors['invalidCreditCard']) return 'Invalid credit card number';
    if (errors['invalidUrl']) return 'Invalid URL format';
    
    return 'Invalid value';
  }
}
```

**Pro Tip:** Create a **centralized validation service** that provides both validators and error message mappings. This ensures consistency across your app and makes it easy to update validation rules globally. Most candidates hard-code validation logic in components, making it difficult to maintain. Also, use **async validators sparingly** - they trigger HTTP requests, so always add debouncing and caching to avoid hammering your API.

---

*Continue reading: [Form Arrays](#form-arrays), [Dynamic Forms](#dynamic-forms), [Custom Validators](#custom-validators)*
