# Twin trim  
  
Trim(  
    With(  
        {  
            // Cut off the text at the start of the citation footer "[1]:"  
            // If no citations exist, this keeps the whole text.  
            CleanedFooter: First(Split(System.Response.FormattedText, "[1]:")).Value  
        },  
        // Remove inline markers [1] through [6]  
        Substitute(  
            Substitute(  
                Substitute(  
                    Substitute(  
                        Substitute(  
                            Substitute(  
                                CleanedFooter,  
                                "[1]", ""  
                            ),  
                            "[2]", ""  
                        ),  
                        "[3]", ""  
                    ),  
                    "[4]", ""  
                ),  
                "[5]", ""  
            ),  
            "[6]", ""  
        )  
    )  
)  
