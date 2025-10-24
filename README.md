package com.suntec.custom.service;

import com.suntec.custom.jaxb.services.types.PasswordRepoService;
import com.suntec.custom.jaxb.services.v10.passwordrepositoryservice.QueryClearTextPassword;
import com.suntec.custom.jaxb.services.v10.passwordrepositoryservice.QueryClearTextPasswordResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.Spy;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.test.util.ReflectionTestUtils;
import org.springframework.ws.WebServiceMessage;
import org.springframework.ws.client.core.WebServiceMessageCallback;
import org.springframework.ws.client.core.WebServiceTemplate;
import org.springframework.ws.soap.SoapHeader;
import org.springframework.ws.soap.SoapMessage;

import jakarta.xml.bind.JAXBElement;
import javax.xml.namespace.QName;
import javax.xml.transform.Result;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class PasswordClientTest {

    @Mock
    private WebServiceTemplate webServiceTemplate;

    @Spy
    private PasswordClient passwordClient;

    private static final String CLEAR_PASSWORD_API = "http://test.api/password";
    private static final String CUSTOMER_ID = "CUST123";

    @BeforeEach
    void setUp() {
        ReflectionTestUtils.setField(passwordClient, "clearPasswordApi", CLEAR_PASSWORD_API);
        doReturn(webServiceTemplate).when(passwordClient).getWebServiceTemplate();
    }

    @Test
    void testGetPassword_Success() {
        // Arrange
        QueryClearTextPasswordResponse expectedResponse = new QueryClearTextPasswordResponse();
        JAXBElement<QueryClearTextPasswordResponse> jaxbElement = new JAXBElement<>(
                new QName("http://test", "response"),
                QueryClearTextPasswordResponse.class,
                expectedResponse
        );

        when(webServiceTemplate.marshalSendAndReceive(
                eq(CLEAR_PASSWORD_API),
                any(QueryClearTextPassword.class),
                any(WebServiceMessageCallback.class)
        )).thenReturn(jaxbElement);

        // Act
        QueryClearTextPasswordResponse result = passwordClient.getPassword(CUSTOMER_ID);

        // Assert
        assertNotNull(result);
        assertEquals(expectedResponse, result);
        verify(webServiceTemplate).marshalSendAndReceive(
                eq(CLEAR_PASSWORD_API),
                any(QueryClearTextPassword.class),
                any(WebServiceMessageCallback.class)
        );
    }

    @Test
    void testGetPassword_RequestBuiltCorrectly() {
        // Arrange
        QueryClearTextPasswordResponse expectedResponse = new QueryClearTextPasswordResponse();
        JAXBElement<QueryClearTextPasswordResponse> jaxbElement = new JAXBElement<>(
                new QName("http://test", "response"),
                QueryClearTextPasswordResponse.class,
                expectedResponse
        );

        ArgumentCaptor<QueryClearTextPassword> requestCaptor = ArgumentCaptor.forClass(QueryClearTextPassword.class);

        when(webServiceTemplate.marshalSendAndReceive(
                eq(CLEAR_PASSWORD_API),
                requestCaptor.capture(),
                any(WebServiceMessageCallback.class)
        )).thenReturn(jaxbElement);

        // Act
        passwordClient.getPassword(CUSTOMER_ID);

        // Assert
        QueryClearTextPassword capturedRequest = requestCaptor.getValue();
        assertNotNull(capturedRequest);
        assertNotNull(capturedRequest.getArg0());
        assertEquals(CUSTOMER_ID, capturedRequest.getArg0().getEntityId());
        assertEquals("C", capturedRequest.getArg0().getEntityType());
    }

    @Test
    void testGetPassword_WithNonJAXBElementResponse() {
        // Arrange
        QueryClearTextPasswordResponse expectedResponse = new QueryClearTextPasswordResponse();

        when(webServiceTemplate.marshalSendAndReceive(
                eq(CLEAR_PASSWORD_API),
                any(QueryClearTextPassword.class),
                any(WebServiceMessageCallback.class)
        )).thenReturn(expectedResponse);

        // Act
        QueryClearTextPasswordResponse result = passwordClient.getPassword(CUSTOMER_ID);

        // Assert
        assertNotNull(result);
        assertEquals(expectedResponse, result);
    }

    @Test
    void testGetPassword_SoapHeaderCallback() throws Exception {
        // Arrange
        QueryClearTextPasswordResponse expectedResponse = new QueryClearTextPasswordResponse();
        JAXBElement<QueryClearTextPasswordResponse> jaxbElement = new JAXBElement<>(
                new QName("http://test", "response"),
                QueryClearTextPasswordResponse.class,
                expectedResponse
        );

        ArgumentCaptor<WebServiceMessageCallback> callbackCaptor = ArgumentCaptor.forClass(WebServiceMessageCallback.class);
        SoapMessage soapMessage = mock(SoapMessage.class);
        SoapHeader soapHeader = mock(SoapHeader.class);
        Result result = mock(Result.class);

        when(soapMessage.getSoapHeader()).thenReturn(soapHeader);
        when(soapHeader.getResult()).thenReturn(result);

        when(webServiceTemplate.marshalSendAndReceive(
                eq(CLEAR_PASSWORD_API),
                any(QueryClearTextPassword.class),
                callbackCaptor.capture()
        )).thenReturn(jaxbElement);

        // Act
        passwordClient.getPassword(CUSTOMER_ID);
        WebServiceMessageCallback callback = callbackCaptor.getValue();
        callback.doWithMessage(soapMessage);

        // Assert
        verify(soapMessage).getSoapHeader();
        verify(soapHeader).getResult();
    }

    @Test
    void testGetPassword_SoapHeaderCallbackWithException() throws Exception {
        // Arrange
        QueryClearTextPasswordResponse expectedResponse = new QueryClearTextPasswordResponse();
        JAXBElement<QueryClearTextPasswordResponse> jaxbElement = new JAXBElement<>(
                new QName("http://test", "response"),
                QueryClearTextPasswordResponse.class,
                expectedResponse
        );

        ArgumentCaptor<WebServiceMessageCallback> callbackCaptor = ArgumentCaptor.forClass(WebServiceMessageCallback.class);
        SoapMessage soapMessage = mock(SoapMessage.class);

        when(soapMessage.getSoapHeader()).thenThrow(new RuntimeException("SOAP Header error"));

        when(webServiceTemplate.marshalSendAndReceive(
                eq(CLEAR_PASSWORD_API),
                any(QueryClearTextPassword.class),
                callbackCaptor.capture()
        )).thenReturn(jaxbElement);

        // Act
        QueryClearTextPasswordResponse result = passwordClient.getPassword(CUSTOMER_ID);
        WebServiceMessageCallback callback = callbackCaptor.getValue();
        
        // This should not throw an exception, it should be caught and logged
        assertDoesNotThrow(() -> callback.doWithMessage(soapMessage));

        // Assert
        assertNotNull(result);
        verify(soapMessage).getSoapHeader();
    }

    @Test
    void testGetPassword_DifferentCustomerId() {
        // Arrange
        String differentCustId = "CUST456";
        QueryClearTextPasswordResponse expectedResponse = new QueryClearTextPasswordResponse();
        JAXBElement<QueryClearTextPasswordResponse> jaxbElement = new JAXBElement<>(
                new QName("http://test", "response"),
                QueryClearTextPasswordResponse.class,
                expectedResponse
        );

        ArgumentCaptor<QueryClearTextPassword> requestCaptor = ArgumentCaptor.forClass(QueryClearTextPassword.class);

        when(webServiceTemplate.marshalSendAndReceive(
                eq(CLEAR_PASSWORD_API),
                requestCaptor.capture(),
                any(WebServiceMessageCallback.class)
        )).thenReturn(jaxbElement);

        // Act
        passwordClient.getPassword(differentCustId);

        // Assert
        QueryClearTextPassword capturedRequest = requestCaptor.getValue();
        assertEquals(differentCustId, capturedRequest.getArg0().getEntityId());
    }

    @Test
    void testGetPassword_NullResponse() {
        // Arrange
        when(webServiceTemplate.marshalSendAndReceive(
                eq(CLEAR_PASSWORD_API),
                any(QueryClearTextPassword.class),
                any(WebServiceMessageCallback.class)
        )).thenReturn(null);

        // Act & Assert
        assertThrows(NullPointerException.class, () -> passwordClient.getPassword(CUSTOMER_ID));
    }
}
