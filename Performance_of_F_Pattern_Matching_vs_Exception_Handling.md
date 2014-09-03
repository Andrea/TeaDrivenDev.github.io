# Performance of F# Pattern Matching versus Exception Handling

I have been asked to share a little finding I made regarding the performance of pattern matching versus exception handling in F#, so here we go.

I was tracking down a performance issue presumably (and also actually) with the following method:

    override this.Equals other =
        match other with
        | null -> false
        | :? DingsWithIdEquality as otherDings -> this.idForEquals = otherDings.idForEquals
        | _ -> false

The IL generated with F# 3.1.2 for this method is this:

    public override bool Equals(object other)
    {
        object obj = other;
        if (obj == null)
          return false;
        main.DingsWithIdEquality dingsWithIdEquality = obj as main.DingsWithIdEquality;
        if (dingsWithIdEquality != null)
          return LanguagePrimitives.HashCompare.GenericEqualityIntrinsic<Guid>(this.idForEquals@, dingsWithIdEquality.idForEquals@);
        else
          return false;
    }
    
This looked complicated and potentially inefficient to my rather untrained eye, and as I was only experimenting and **knew** there really wouldn't be any comparisons to other types, I tried simplifying this a little:

    override this.Equals other =
        try
            let otherDings = other :?> DingsWithIdEquality
            otherDings.idForEquals.Equals(this.idForEquals)
        with
            _ -> false
            
Sure enough, the IL looks much shorter:

    public override bool Equals(object other)
    {
        try
        {
          return LanguagePrimitives.HashCompare.GenericEqualityIntrinsic<Guid>(LanguagePrimitives.IntrinsicFunctions.UnboxGeneric<main.DingsWithIdEquality>(other).idForEquals@, this.idForEquals@);
        }
        catch
        {
          return false;
        }
    }
    
But here's the.... catch - it's slower, even if the `catch` is never actually executed. Finding 5,000 items in an array of 300,000 using the `Equals()` method that uses exception handling took about 20% longer than with pattern matching on the type, even if no exception ever occurred in that `try` block.


For completeness, here's the IL for the idiomatic C# equivalent, which is almost exactly the real C# code and to me looks like the most efficient way to really handle the task:

    public override bool Equals(object obj)
    {
      DingsIdEquality dingsIdEquality = obj as DingsIdEquality;
      return dingsIdEquality != null && this.id == dingsIdEquality.id;
    }


I'll also say that of course that was not the issue I was trying to get at; the actual problem was a bit more subtle.