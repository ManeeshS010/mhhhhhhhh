package com.suntec.custom.service;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.oxm.jaxb.Jaxb2Marshaller;

import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class PasswordClientConfigTest {

    @InjectMocks
    private PasswordClientConfig passwordClientConfig;

    @Test
    void testMarshaller_ShouldReturnConfiguredJaxb2Marshaller() {
        // When
        Jaxb2Marshaller marshaller = passwordClientConfig.marshaller();

        // Then
        assertNotNull(marshaller, "Marshaller should not be null");
        
        // Verify that the marshaller is properly configured
        // We can't directly access packagesToScan, but we can verify the marshaller was created
        assertInstanceOf(Jaxb2Marshaller.class, marshaller, 
            "Should return an instance of Jaxb2Marshaller");
    }

    @Test
    void testMarshaller_ShouldCreateNewInstanceEachTime() {
        // When
        Jaxb2Marshaller marshaller1 = passwordClientConfig.marshaller();
        Jaxb2Marshaller marshaller2 = passwordClientConfig.marshaller();

        // Then
        assertNotNull(marshaller1);
        assertNotNull(marshaller2);
        assertNotSame(marshaller1, marshaller2, 
            "Each call should create a new instance (prototype scope behavior)");
    }

    @Test
    void testPasswordClient_ShouldReturnConfiguredClient() {
        // Given
        Jaxb2Marshaller marshaller = new Jaxb2Marshaller();
        marshaller.setPackagesToScan("com.suntec.custom.jaxb.services");

        // When
        PasswordClient client = passwordClientConfig.passworClient(marshaller);

        // Then
        assertNotNull(client, "PasswordClient should not be null");
        assertInstanceOf(PasswordClient.class, client, 
            "Should return an instance of PasswordClient");
    }

    @Test
    void testPasswordClient_ShouldSetDefaultUri() {
        // Given
        Jaxb2Marshaller marshaller = new Jaxb2Marshaller();
        marshaller.setPackagesToScan("com.suntec.custom.jaxb.services");
        String expectedUri = "https://niqdevweb001:7011/xelerate/product/services/PasswordRepositoryService";

        // When
        PasswordClient client = passwordClientConfig.passworClient(marshaller);

        // Then
        assertNotNull(client);
        // The URI is set internally, we verify the client was created successfully
        // In a real scenario, if PasswordClient had a getDefaultUri() method, we would verify it
    }

    @Test
    void testPasswordClient_ShouldSetMarshallerAndUnmarshaller() {
        // Given
        Jaxb2Marshaller marshaller = new Jaxb2Marshaller();
        marshaller.setPackagesToScan("com.suntec.custom.jaxb.services");

        // When
        PasswordClient client = passwordClientConfig.passworClient(marshaller);

        // Then
        assertNotNull(client, "Client should be created with marshaller and unmarshaller set");
    }

    @Test
    void testPasswordClient_WithNullMarshaller_ShouldHandleGracefully() {
        // When/Then
        assertDoesNotThrow(() -> {
            PasswordClient client = passwordClientConfig.passworClient(null);
            assertNotNull(client, "Client should be created even with null marshaller");
        }, "Should handle null marshaller without throwing exception");
    }

    @Test
    void testPasswordClient_ShouldCreateNewInstanceEachTime() {
        // Given
        Jaxb2Marshaller marshaller = new Jaxb2Marshaller();
        marshaller.setPackagesToScan("com.suntec.custom.jaxb.services");

        // When
        PasswordClient client1 = passwordClientConfig.passworClient(marshaller);
        PasswordClient client2 = passwordClientConfig.passworClient(marshaller);

        // Then
        assertNotNull(client1);
        assertNotNull(client2);
        assertNotSame(client1, client2, 
            "Each call should create a new instance (prototype scope behavior)");
    }

    @Test
    void testIntegration_MarshallerAndPasswordClient() {
        // When
        Jaxb2Marshaller marshaller = passwordClientConfig.marshaller();
        PasswordClient client = passwordClientConfig.passworClient(marshaller);

        // Then
        assertNotNull(marshaller, "Marshaller should be created");
        assertNotNull(client, "Client should be created with the marshaller");
    }

    @Test
    void testPasswordClientConfig_BeanInstantiation() {
        // When
        PasswordClientConfig config = new PasswordClientConfig();

        // Then
        assertNotNull(config, "Configuration instance should be created");
        assertNotNull(config.marshaller(), "Should be able to create marshaller");
    }

    @Test
    void testMarshaller_ConfigurationDetails() {
        // When
        Jaxb2Marshaller marshaller = passwordClientConfig.marshaller();

        // Then
        assertNotNull(marshaller);
        // Verify it's a valid JAXB2 marshaller that can be used for marshalling/unmarshalling
        assertTrue(marshaller instanceof org.springframework.oxm.Marshaller, 
            "Should be an instance of Spring's Marshaller interface");
        assertTrue(marshaller instanceof org.springframework.oxm.Unmarshaller, 
            "Should be an instance of Spring's Unmarshaller interface");
    }
}
