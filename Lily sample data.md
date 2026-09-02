# Lily sample data  
  
kind: Record  
properties:  
  attachmentSizes:  
    type:  
      kind: Table  
  clientActivityID: String  
  enableDiagnostics: Boolean  
  testMode: String  
  triggerTest:  
    type:  
      kind: Record  
      properties:  
        flowId: String  
        flowRunId: String  
        payload: String  
        trigger:  
          type:  
            kind: Record  
            properties:  
              connectorDisplayName: String  
              connectorIconUri: String  
              displayName: String  
              id: String  
        triggerType: String  
        version: String  
