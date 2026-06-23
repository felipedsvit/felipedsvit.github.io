---
title: "Go: hash e verificação de senha com Argon2id"
tags: [go, auth, segurança]
date: 2026-01-20
---

Parâmetros seguros para Argon2id e validação de complexidade de senha. Extraído do [Erreia](https://github.com/felipedsvit/erreia/blob/main/internal/auth/auth.go).

```go
import "github.com/alexedwards/argon2id"

var params = &argon2id.Params{
    Memory:      64 * 1024, // 64 MB
    Iterations:  3,
    Parallelism: 1,
    SaltLength:  16,
    KeyLength:   32,
}

// Hash gera o hash Argon2id com salt aleatório.
// O resultado inclui o salt — armazene o retorno inteiro no banco.
func Hash(password string) (string, error) {
    return argon2id.CreateHash(password, params)
}

// Verify compara senha em texto plano com o hash armazenado.
// Retorna false (não erro) para credenciais inválidas.
func Verify(password, hash string) (bool, error) {
    return argon2id.ComparePasswordAndHash(password, hash)
}
```

Validação de complexidade antes do hash:

```go
func validatePassword(password string) error {
    if len(password) < 8 || len(password) > 128 {
        return errors.New("senha deve ter entre 8 e 128 caracteres")
    }
    var hasUpper, hasLower, hasDigit bool
    for _, ch := range password {
        switch {
        case unicode.IsUpper(ch):
            hasUpper = true
        case unicode.IsLower(ch):
            hasLower = true
        case unicode.IsDigit(ch):
            hasDigit = true
        }
    }
    if !hasUpper || !hasLower || !hasDigit {
        return errors.New("senha precisa de maiúscula, minúscula e dígito")
    }
    return nil
}
```

> `lib: github.com/alexedwards/argon2id`
