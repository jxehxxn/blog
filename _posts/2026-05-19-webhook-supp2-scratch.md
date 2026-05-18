---
layout: post
title: "Webhook 보충 2: Scratch HTTP Server"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook supplement
---

```go
http.HandleFunc("/validate", func(w http.ResponseWriter, r *http.Request) {
    var review admissionv1.AdmissionReview
    json.NewDecoder(r.Body).Decode(&review)
    
    response := admissionv1.AdmissionResponse{ UID: review.Request.UID, Allowed: true }
    if shouldDeny(review.Request) {
        response.Allowed = false
        response.Result = &metav1.Status{Message: "denied"}
    }
    review.Response = &response
    json.NewEncoder(w).Encode(review)
})
```

직접 짜기. controller-runtime 없이.

## 결론
간단 webhook은 직접 가능. complex는 framework.
