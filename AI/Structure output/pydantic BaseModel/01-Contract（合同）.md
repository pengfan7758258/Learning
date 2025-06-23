```python
class Contract(BaseModel):
    """
    Represents the key details of the contract.
    """
    summary: str = Field(
        ...,
        description=("High level summary of the contract with relevant facts and details. Include all relevant information to provide full picture."
        "Do no use any pronouns"),
    )
    contract_type: str = Field(
        ...,
        description="The type of contract being entered into.",
        enum=CONTRACT_TYPES,
    )
    parties: List[Organization] = Field(
        ...,
        description="List of parties involved in the contract, with details of each party's role.",
    )
    effective_date: str = Field(
        ...,
        description=(
          "Enter the date when the contract becomes effective in yyyy-MM-dd format."
          "If only the year (e.g., 2015) is known, use 2015-01-01 as the default date."
          "Always fill in full date"
    )
    contract_scope: str = Field(
        ...,
        description="Description of the scope of the contract, including rights, duties, and any limitations.",
    )
    duration: Optional[str] = Field(
        None,
        description=(
          "The duration of the agreement, including provisions for renewal or termination."
          "Use ISO 8601 durations standard"
        ),
    )
    end_date: Optional[str] = Field(
        None,
        description=(
            "The date when the contract expires. Use yyyy-MM-dd format."
            "If only the year (e.g., 2015) is known, use 2015-01-01 as the default date."
            "Always fill in full date"
        ),
    )
    total_amount: Optional[float] = Field(
        None, description="Total value of the contract."
    )
    governing_law: Optional[Location] = Field(
        None, description="The jurisdiction's laws governing the contract."
    )
    clauses: Optional[List[Clause]] = Field(
        None, description=f"""Relevant summaries of clause types. Allowed clause types are {CLAUSE_TYPES}"""
    )
```
