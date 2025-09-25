# Сериализация

```
DeserializeOwned
```

```
fn parse_jwt_token<T>(token: &str) -> anyhow::Result<T> where T: DeserializeOwned {
    use base64::prelude::*;
    use itertools::Itertools;
    let (_header, body, _sign) = token.split(".").collect_tuple().unwrap();
    let decoded = BASE64_URL_SAFE_NO_PAD.decode(body).unwrap();
    let json_text = String::from_utf8(decoded).unwrap();
    let payload: T = serde_json::from_str(&json_text).unwrap();
    Ok(payload)
}
```

