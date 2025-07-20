```python
class Location(BaseModel):
    """
    Represents a physical location including address, city, state, and country.
    """
    address: Optional[str] = Field(
        ..., description="The street address of the location.Use None if not provided"
    )
    city: Optional[str] = Field(
        ..., description="The city of the location.Use None if not provided"
    )
    state: Optional[str] = Field(
        ..., description="The state or region of the location.Use None if not provided"
    )
    country: str = Field(
        ...,
        description="The country of the location. Use the two-letter ISO standard.",
    )
```
- `country` 字段遵循两位国家代码标准。如 `"US"`、 `"FR"` 或 `"JP"`，而不是不一致的变体，如 "United States" 或 "USA"。这一原则也适用于其他结构化数据，例如 ISO 8601 保持日期格式标准化（`YYYY-MM-DD`）